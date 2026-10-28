---
title: After a compute reboot, instances stay SHUTOFF or come back without network
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, neutron, ovs, libvirt]
deployment_methods: [kolla-ansible]
environment:
  topology: any multinode, a compute rebooted (planned or not)
  network: ml2/ovs
  storage: any
  os: any
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, all-in-one", date: 2026-10-24, note: "cause 1 reproduced by rebooting the host, with and without resume_guests_state_on_host_boot; cause 3 by taking the external interface down; causes 2, 4 and 5 not reproduced" }
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#DEFAULT.resume_guests_state_on_host_boot
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/networking/neutron.html
  - https://docs.openstack.org/neutron/2026.1/admin/config-mtu.html
authored_by: agent:ozarland-author
last_verified: 2026-10-24
---

# After a compute reboot, instances stay SHUTOFF or come back without network

## Symptom

A compute was rebooted: a kernel update, a hardware intervention, or a crash. The host is back and answers SSH.
Then one of two things:

- its instances are `SHUTOFF` and stay so, although they were running before the reboot;
- the instances run, but they have no network: no ping, no SSH, while Nova shows them `ACTIVE`.

```text
$ openstack server list --all-projects --host <compute> --long -c Name -c Status -c "Power State"
| <instance> | SHUTOFF | Shutdown |
```

## Impact

Every instance of the rebooted compute, until someone acts. Nothing is lost on disk: a `SHUTOFF` instance keeps its
disks, and an instance without network keeps running.

## Quick checks

From any host with the client, with admin credentials:

```bash
# 1. State of the compute's instances, and of its services
openstack server list --all-projects --host <compute> --long -c Name -c Status -c "Power State"
openstack compute service list --host <compute>
openstack network agent list --host <compute>
```

On the compute:

**Kolla-Ansible:**

```bash
# 2. Did every container unit come back?
systemctl list-units 'kolla-*' --all --no-pager --state=failed,inactive,activating
sudo docker ps -a --filter name=openvswitch --format '{{.Names}}\t{{.Status}}'

# 3. Does Nova start the guests that ran before a reboot? (empty output: the default, False)
sudo docker exec nova_compute grep -rn resume_guests_state_on_host_boot /etc/nova/

# 4. Are the interfaces up, with the expected MTU?
ip -br link
ip link show <tunnel-interface> | grep -o 'mtu [0-9]*'
ip link show <neutron-external-interface>

# 5. Is the external interface plugged into the external bridge?
sudo docker exec openvswitch_db ovs-vsctl list-ports br-ex
```

## Diagnosis

### Nova

`resume_guests_state_on_host_boot` "specifies whether to start guests that were running before the host rebooted"
(Nova configuration reference). Its default is `False`, and Kolla-Ansible 2026.1 does not set it. With the default,
the guests that ran before the reboot are not started again, and Nova shows them `SHUTOFF` once it has
synchronised their power state. This is expected
behaviour, not a failure.

Until `nova-compute` is back and has synchronised, Nova still shows the instances `ACTIVE` and `Running`, from before
the reboot. Read quick check 1 only once the compute service is `up` again. On the lab, where the controller and the
compute were the same host, the instances switched to `SHUTOFF` about a minute after `nova-compute` was back.

With the option set to `true`, `nova-compute` starts them at its own start, and logs for each one
`Rebooting instance after nova-compute restart.`

### Kolla-Ansible containers

In Kolla-Ansible 2026.1 each container runs from a systemd unit (`kolla-<container>-container.service`) that is
`WantedBy=multi-user.target` and ordered after `docker.service` only (the unit template in Kolla-Ansible's
`kolla_systemd_worker`). The units come back at boot by themselves. A unit that failed to start (for example because
an interface it needs was not up yet) shows in quick check 2.

Kolla-Ansible does not remove the unit of a service that was disabled after it was deployed (for example
`enable_haproxy: "no"` set later). Such a unit also shows in quick check 2, `failed` or `activating auto-restart`,
at every boot. Compare with the services enabled in `globals.yml` before treating it as the cause.

### Host network

Kolla-Ansible does not configure the host's interfaces, bonds, VLANs or MTU: the host's own network configuration
does, at boot. Anything set by hand before the reboot (`ip link set ... mtu ...`, `ip link set ... up`, an address
moved by hand) is lost.

- **An external interface or bond not up:** `br-ex` has no uplink; floating IPs and provider networks of this host
  stop answering.
- **A non-persistent MTU:** the tunnel interface comes back at its default MTU, smaller than before. Small packets pass,
  large ones are dropped: see [tunnel MTU mismatch](../networking/tunnel-mtu-mismatch-large-packets-dropped.md).
- **`br-ex` and addresses:** Kolla-Ansible plugs `neutron_external_interface` into `br-ex`, and its Neutron guide
  recommends leaving that interface without an IP address. A host address placed by hand on `br-ex`, or left on the
  plugged interface, does not survive the reboot as intended: the host itself can lose that network.

### Open vSwitch

If `openvswitch_vswitchd` did not come back, every instance of the compute is without network: see
[ovs-vswitchd down on a compute](../networking/ovs-vswitchd-down-on-compute.md).

## Possible causes

| Cause | How to tell |
|---|---|
| 1. Nova does not resume guests after a reboot (the default) | Instances `SHUTOFF`; quick check 3 empty or `False` |
| 2. A Kolla-Ansible container did not come back | Quick check 2 lists a unit not running; `openvswitch_vswitchd` among them cuts every instance |
| 3. The external interface, bond or VLAN is not up | Quick check 4: `DOWN`; floating IPs and provider networks of this host do not answer |
| 4. The MTU set by hand was not persistent | Quick check 4: a smaller MTU than the other computes; ping works, large transfers stall |
| 5. An address was on `br-ex` or on the plugged interface | The host lost an address after the reboot; quick check 5 |

## Resolution

### For cause 1: start the instances, then decide the policy

> ⚠️ **State-changing.** Starts the guest. Check with its owner that it was meant to run: an instance stopped on
> purpose before the reboot is also `SHUTOFF`.

```bash
openstack server start <server-id>
```

To make Nova start them by itself after the next reboot, set the option through an override. This file applies it to
the computes only; the global `/etc/kolla/config/nova.conf` works too:

```ini
# /etc/kolla/config/nova/nova-compute.conf
[DEFAULT]
resume_guests_state_on_host_boot = true
```

> ⚠️ **State-changing.** `reconfigure` restarts the Nova containers whose configuration changed; running instances
> are not affected. Undo: remove the override and reconfigure again.

```bash
kolla-ansible reconfigure -i <inventory> --tags nova
```

### For cause 2: start the unit, then find why it did not start

*(Cause 2 not reproduced on this page's lab.)*

> ⚠️ **State-changing.** Starts the container. For `openvswitch_vswitchd`, the instances regain their network about
> half a minute later on the lab of the ovs-vswitchd runbook.

```bash
sudo systemctl start kolla-<container>-container.service
journalctl -u kolla-<container>-container.service -b --no-pager | tail -n 50
```

### For causes 3 and 4: fix the host's network configuration, persistently

*(Run on the lab for cause 3; cause 4 not reproduced.)*

Put the interface, bond, VLAN and MTU in the host's network configuration (netplan, NetworkManager,
systemd-networkd), so that the next reboot brings them back. To restore service now:

> ⚠️ **State-changing.** Changes the interface live. A wrong MTU or interface name can cut the host's own
> connectivity: have console access, and change one interface at a time.

```bash
sudo ip link set dev <interface> up
sudo ip link set dev <tunnel-interface> mtu <mtu-used-by-the-other-computes>
```

### For cause 5: the address on the external interface

*(Not run on the lab.)*

Restore the host's address from the console, in the host's network configuration, on an interface that is not
plugged into `br-ex`. Kolla-Ansible's Neutron guide calls a shared external interface (an address and the bridge on
the same interface) "out of scope" for Kolla-Ansible; this runbook does not cover setting one up.

## Verification

```bash
openstack server list --all-projects --host <compute> --long -c Name -c Status -c "Power State"   # ACTIVE, Running
openstack network agent list --host <compute>                                                      # all alive
ping -c 3 <instance-floating-ip>
```

On the compute, quick checks 2, 4 and 5 show every unit running, every interface up with the expected MTU, and the
external interface plugged into `br-ex`.

## Lab observations

On a lab: OpenStack 2026.1, Kolla-Ansible 22.2.0, all-in-one (controller and compute on one VM), ML2/OVS, Docker,
Ubuntu 24.04, two CirrOS instances with floating IPs. The whole host was rebooted twice.

| | Reboot 1, option not set (default) | Reboot 2, option set to `true` |
|---|---|---|
| Quick check 3 | Empty | `resume_guests_state_on_host_boot = true` |
| Instances, as Nova showed them right after the boot | `ACTIVE`, `Running` (not yet synchronised) | `ACTIVE`, `Running` |
| Instances once `nova-compute` was up | **`SHUTOFF`, `Shutdown`**, about a minute later | `ACTIVE`, `Running`; both domains `running` in `virsh list`; `Rebooting instance after nova-compute restart.` in `nova-compute.log` |
| Floating IPs | No answer until `openstack server start` | Answered |
| Quick checks 2, 4, 5 | Every unit back, except two units of services disabled after the deployment (see Diagnosis); interfaces up; the external interface in `br-ex` | Same |

- `openstack server start` brought both instances back; their floating IPs answered.
- The override file `/etc/kolla/config/nova/nova-compute.conf` and `kolla-ansible reconfigure -i <inventory> --tags nova`
  worked as written. Only `nova_compute` was recreated, and running instances were not affected. Removing the file
  and reconfiguring again undid it.
- Cause 3, without a reboot: with the external interface set `DOWN`, quick check 4 showed it `DOWN` and the floating
  IPs stopped answering. `ip link set dev <interface> up` restored them.
- On this all-in-one, the control plane restarted with the host. Keystone answered about 8 to 10 minutes after the
  boot: every container started at once on 4 vCPU. A compute that is not a controller comes back sooner.

## Validity and limits

Written from the Nova 2026.1 configuration reference (`resume_guests_state_on_host_boot` and its default), the
Kolla-Ansible 2026.1 source (the systemd unit template, and no setting of that option) and its Neutron guide, and
Neutron's MTU guide. Observed on one all-in-one lab (see Lab observations): cause 1 with and without the option, and
cause 3 without a reboot. Not observed: causes 2, 4 and 5, and a reboot of a compute of a multinode platform, where
the controllers stay up during the reboot.

Not covered: ML2/OVN (the gateway chassis and `ovn-controller` behave differently after a reboot), DPDK, instances
whose disks are on Ceph, an unplanned host failure where the instances must be evacuated (see
[fence before you evacuate](compute-host-failure-fence-before-evacuate.md)).

## Related

- [ovs-vswitchd down on a compute](../networking/ovs-vswitchd-down-on-compute.md)
- [Tunnel MTU mismatch, large packets dropped](../networking/tunnel-mtu-mismatch-large-packets-dropped.md)
- [nova-compute down, instances still running](nova-compute-down-instances-still-running.md)
- [Moving instances off a host for maintenance](../migration/moving-instances-off-a-host-for-maintenance.md): before a
  planned reboot
- Nova, [`resume_guests_state_on_host_boot`](https://docs.openstack.org/nova/2026.1/configuration/config.html#DEFAULT.resume_guests_state_on_host_boot)
- Kolla-Ansible 2026.1, [Neutron guide](https://docs.openstack.org/kolla-ansible/2026.1/reference/networking/neutron.html)

---

*Provenance: written by agent:ozarland-author, from upstream documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=recovery/after-a-compute-reboot-instances-shutoff-or-without-network.md&title=Feedback%3A+recovery/after-a-compute-reboot-instances-shutoff-or-without-network.md):
your release, your deployment method, and what happened.
