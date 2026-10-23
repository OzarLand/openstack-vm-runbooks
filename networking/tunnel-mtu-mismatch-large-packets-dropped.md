---
title: Instances of one compute answer ping, but large transfers stall
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [neutron, ovs]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers, 2 computes
  network: ml2/ovs (VXLAN)
  storage: none (no Cinder on the lab)
  os: ubuntu-24.04 (noble), docker
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible multinode (1 controller, 2 computes)", date: 2026-10-23, note: "cause 1 reproduced on one compute and fixed with ip link set, persisted with netplan apply; the host-to-host DF ping from the other host did not detect the lowered MTU" }
upstream_refs:
  - https://docs.openstack.org/neutron/latest/admin/config-mtu.html
  - https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html
  - https://docs.openstack.org/kolla-ansible/latest/admin/advanced-configuration.html
last_verified: 2026-10-23
---

# Instances of one compute answer ping, but large transfers stall

The interface that carries the VXLAN tunnels of one compute has a smaller MTU than Neutron assumes. Small packets of
the instances still fit once encapsulated; full-size packets do not, and are dropped without any error. Ping and most
monitoring stay green.

## Symptom

- The instances of one compute answer ping and accept connections, but transfers stall: SSH logs in and then hangs
  on a large output, HTTP requests with a large answer time out, TLS handshakes with a long certificate chain fail.
- Instances on the other computes are fine.
- Nova and Neutron report nothing wrong: agents alive, instances `ACTIVE`, no error in the logs.

A ping with "do not fragment" at the largest payload of the instance network shows it:

```text
$ ping -M do -s 1364 -c 3 <instance-ip>      # instance on compute-1, network MTU 1392
3 packets transmitted, 0 received, 100% packet loss

$ ping -M do -s 56 -c 3 <instance-ip>        # same instance, small packets
3 packets transmitted, 3 received, 0% packet loss
```

## Impact

Every instance on the affected compute, for every packet larger than what still fits the tunnel. Interactive and
small-packet traffic keeps working, so users report "slow" or "hanging" services rather than an outage. Instances on
other computes are not affected, except when they talk to the affected compute. No data is at risk.

## Quick checks

The largest ICMP payload is the network MTU less 28 bytes (20 of IPv4 header, 8 of ICMP).

```bash
# 1. On a controller, admin credentials: the MTU Neutron gives the instance network
openstack network show <network> -c mtu -c provider:network_type

# 2. On a network node: "do not fragment" pings from the router namespace to an instance on the suspect compute,
#    then to an instance on another compute. Default size, then the largest payload (<network MTU - 28>).
ip netns list | grep qrouter
ip netns exec qrouter-<router-id> ping -M do -s 56 -c 3 <instance-ip>
ip netns exec qrouter-<router-id> ping -M do -s <network-mtu-minus-28> -c 3 <instance-ip>

# 3. On each compute: the MTU of the interface that carries the tunnels
ip link show <tunnel-interface>
```

This page applies when the small ping answers, the largest one does not, and only for instances of one compute.
That compute's tunnel interface MTU is then smaller than the network MTU plus 50 bytes (VXLAN over IPv4).

## Diagnosis

### Which interface carries the tunnels

The tunnels leave the compute from the interface that holds the agent's `local_ip`.

**Kolla-Ansible:** on the compute, the address is in the agent configuration, and the VXLAN ports show it too:

```bash
grep -E '^\s*local_ip' /etc/kolla/neutron-openvswitch-agent/openvswitch_agent.ini
docker exec openvswitch_vswitchd ovs-vsctl show | grep -A2 'type: vxlan'
ip -o addr show | grep '<local-ip>'          # gives the interface name
```

In Kolla-Ansible this is `tunnel_interface`, which defaults to `network_interface`.

**Any:** `local_ip` in the `[ovs]` section of `openvswitch_agent.ini`, then `ip -o addr show` as above.

### Compare the computes

```bash
# On every compute
ip link show <tunnel-interface> | grep -o 'mtu [0-9]*'
```

All computes should carry the same value, and it must be at least the largest VXLAN network MTU plus 50. A compute
with a smaller value is the one whose instances lose large packets.

### What Neutron assumes

**Kolla-Ansible:** on a controller:

```bash
grep -E '^\s*global_physnet_mtu' /etc/kolla/neutron-server/neutron.conf
grep -E '^\s*path_mtu' /etc/kolla/neutron-server/ml2_conf.ini
```

Neutron sets the MTU of a VXLAN network to `global_physnet_mtu` (1500 if unset; `path_mtu` if smaller and set) less
50 bytes, and hands it to the instances through DHCP. Kolla-Ansible does not derive it from the hosts: it comes from a
`neutron.conf` override, or from the default. Neutron never re-reads the MTU of the hosts after deployment.

### The underlay between the hosts

The interface MTU of the hosts can be right while a switch on the path drops large frames. Test the path itself,
between the tunnel addresses of two hosts:

```bash
# On compute-1, towards the tunnel address of compute-2
ping -M do -s <tunnel-mtu-minus-28> -c 3 -I <local-ip> <remote-local-ip>
```

## Possible causes

### Cause 1: the tunnel interface MTU of one compute was lowered after deployment

A network change, a NIC or bond reconfigured, an MTU delivered by DHCP on the underlay. `ip link show` gives a smaller
value on that compute than on the others. This is the case the lab reproduced.

### Cause 2: the underlay path drops large frames

All hosts show the same, correct interface MTU, but the host-to-host "do not fragment" ping above fails at the full
tunnel size. A switch or a link on the path has a smaller MTU. Not reproduced on the lab.

### Cause 3: Neutron assumes more than the underlay ever carried

Every compute is affected the same way, not one. `global_physnet_mtu` (or its default of 1500) is larger than the
real interface MTU of all hosts. Not reproduced on the lab.

### Cause 4: IPv6 underlay

With an IPv6 underlay (`tunnel_address_family: ipv6` in Kolla-Ansible) VXLAN takes 70 bytes, not 50. The comparison
above must use 70. Not tested on the lab.

## Resolution

### For cause 1: restore the interface MTU, then make it persistent

> ⚠️ **State-changing.** Changes the MTU of the tunnel interface of one compute. Traffic of its instances recovers at
> once. If the interface also carries management or API traffic, a wrong value can cut it: run it from the out-of-band
> console if in doubt. Undo: set the previous value again with the same command. The change does not survive a reboot.

```bash
# On the affected compute; <mtu> is the value of the other computes
ip link set dev <tunnel-interface> mtu <mtu>
ip link show <tunnel-interface> | grep -o 'mtu [0-9]*'
```

Then make the value persistent in the host's network configuration, so a reboot or a DHCP renewal does not bring the
problem back. With netplan, set `mtu: <mtu>` on the interface in `/etc/netplan/<file>.yaml`.

> ⚠️ **State-changing.** `netplan apply` reconfigures the host's interfaces and can briefly interrupt every network
> of the compute, instance tunnels and management included. Run it from the out-of-band console, in a maintenance
> window. Undo: restore the previous file and apply again.

```bash
netplan apply
```

Then find out who changed the MTU, or the change may come back.

### For cause 2: fix the underlay

Fix the MTU of the switch ports or links on the path. This is outside OpenStack. Until it is fixed, the path carries
less than Neutron assumes, and cause 3 applies.

### For cause 3, or when the smaller MTU is the new reality

Lower what Neutron assumes so that it matches what the underlay carries end to end.

> ⚠️ **State-changing.** Changes the MTU of every tenant network, and so of every instance. Instances pick up the new
> MTU only at their next DHCP renewal or reboot; until then, instances with the old and the new MTU coexist. Decide it
> with the owners of the platform, in a maintenance window. Undo: set the previous values again and reconfigure.

**Kolla-Ansible:** on the deploy host, set the value in the `neutron.conf` override:

```ini
# /etc/kolla/config/neutron.conf
[DEFAULT]
global_physnet_mtu = <underlay-mtu>
```

```bash
kolla-ansible reconfigure -i <inventory> -t neutron
```

Existing networks keep the MTU stored when they were created. Update each one (on a controller, admin credentials):

```bash
openstack network set --mtu <underlay-mtu-minus-50> <network>
```

**Any:** set `global_physnet_mtu` in `neutron.conf` on the Neutron servers, restart `neutron-server`, then update
the existing networks as above.

## Verification

```bash
# On every compute: the same MTU, at least the largest VXLAN network MTU plus 50
ip link show <tunnel-interface> | grep -o 'mtu [0-9]*'

# On a network node: the largest "do not fragment" ping answers, for an instance on every compute
ip netns exec qrouter-<router-id> ping -M do -s <network-mtu-minus-28> -c 3 <instance-ip>
```

After a reboot of the compute, check the MTU again: it must still be the right one.

## Lab observations

On the lab the tunnel interface is the API interface of each host. Its MTU was 1442, so the VXLAN network MTU was
1392 and the largest ICMP payload 1364 bytes. The failure was produced by lowering the MTU of that interface on
compute-1 by 100 bytes, as a network change nobody told Neutron about would.

| | Before | During (90 s) | After (restore + 60 s) |
|---|---|---|---|
| Tunnel MTU compute-1 / compute-2 | 1442 / 1442 | **1342** / 1442 | 1442 / 1442 |
| DF pings to the instance on compute-1 (56 / 1214 / 1364 bytes) | answer / answer / answer | answer / answer / **no answer** | answer / answer / answer |
| DF pings to the instance on compute-2 | answer ×3 | answer ×3 | answer ×3 |
| Continuous 56-byte ping to the instances of compute-1 and compute-2 | – | 191 requests, 0 lost | 378 requests, 0 lost |

- The 1214-byte ping still fits (1242 + 50 = 1292 bytes, under 1342); the 1364-byte one does not
  (1392 + 50 = 1442, over 1342).
- Traffic was back as soon as the MTU was restored with `ip link set`.
- Agents and instances stayed healthy throughout. Neutron and Nova logged nothing.
- The same figures were observed on two separate lab builds of the same configuration. On one of them the "during"
  observation came about 60 s after the change instead of 90 s; the failure showed the same way.

## Validity and limits

- Observed on lab builds with the configuration above: OpenStack 2026.1, Kolla-Ansible 22.2.0, ML2/OVS with VXLAN
  over an IPv4 underlay, Docker, Ubuntu Noble, 3 controllers and 2 computes.
- Observed: the symptom, the signature (one compute, smaller tunnel MTU, largest DF ping lost, small ones fine,
  nothing in the logs), and the fix by restoring the interface MTU with `ip link set`.
- Not run: making the MTU persistent with netplan, the `global_physnet_mtu` change and `openstack network set --mtu`,
  the host-to-host underlay ping. Causes 2, 3 and 4 were not reproduced.
- Not tested: OVN (Geneve has a different overhead), an IPv6 underlay, VLAN or flat networks, other releases, other
  deployment tools, production scale, jumbo frames.
- An MTU set with `ip link` is lost at reboot, and may be overwritten by the host's network manager on a DHCP renewal.

## Related

- [Restarting the OVS agent does not cut instances](restarting-the-ovs-agent-does-not-cut-instances.md)
- [ovs-vswitchd down on a compute](ovs-vswitchd-down-on-compute.md)
- [Control plane down: what still works for running instances](../diagnostics/control-plane-down-what-still-works-for-instances.md)
- Neutron: [MTU considerations](https://docs.openstack.org/neutron/latest/admin/config-mtu.html)
- Kolla-Ansible: [Neutron networking](https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html),
  [service configuration overrides](https://docs.openstack.org/kolla-ansible/latest/admin/advanced-configuration.html)
- Source: lab reproduction by the OzarLand maintainer (OpenStack 2026.1, Kolla-Ansible 22.2.0).
