---
title: Instance build or live migration fails with "Binding failed for port". Neutron could not bind the port on that host
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [neutron, nova, ovs, ovn]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any multinode
  network: ml2/ovs or ml2/ovn
  storage: any
  os: any
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/neutron/2026.1/admin/config-ml2.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/networking/neutron.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Instance build or live migration fails with "Binding failed for port". Neutron could not bind the port on that host

## Symptom

A new instance goes to `ERROR`, or a live migration fails at once, and the `nova-compute` log of the host concerned
says:

```text
Binding failed for port <port-id>, please check neutron logs for more information.
```

The port itself shows that Neutron refused to bind it on that host:

```text
$ openstack port show <port-id> -c binding_vif_type -c binding_host_id -c binding_vnic_type
| binding_host_id   | <compute>      |
| binding_vif_type  | binding_failed |
| binding_vnic_type | normal         |
```

## Impact

Every instance that needs a port on that network and that host: new builds there fail, and live migrations to that
host fail before they start. Often one compute, or one network (a provider network) on some computes. Running
instances whose ports are already bound keep their network.

## Quick checks

Read-only. With admin credentials:

```bash
# 1. The port: where Nova asked to bind it, and the result
openstack port show <port-id> -c binding_host_id -c binding_vif_type -c binding_vnic_type -c network_id

# 2. The network's type and physical network
openstack network show <network-id> -c provider:network_type -c provider:physical_network -c segments

# 3. The agents on that host: present, alive, under the same host name
openstack network agent list --host <compute>
```

On a controller, the Neutron server's log for the port:

**Kolla-Ansible:** `sudo grep <port-id> /var/log/kolla/neutron/neutron-server.log | grep -iE 'bind|refusing'`

**Any:** the same `grep` on the `neutron-server` log.

## Diagnosis

### How a port is bound (Neutron 2026.1, ML2)

Nova sets `binding:host_id` to the compute's host name when it allocates or moves the port. The Neutron server then
asks each mechanism driver to bind the port on that host, for one of the network's segments
(`neutron/plugins/ml2/managers.py`). If none succeeds, it sets `binding:vif_type` to `binding_failed` and logs:

```text
Failed to bind port <port-id> on host <compute> for vnic_type normal using segments [...]
```

Nova reads `binding_failed` and raises `PortBindingFailed` (`nova/network/neutron.py`). For a build, the build fails
(the fault names a failed network allocation, or the build is retried elsewhere until `Exceeded maximum number of
retries`); for a live migration, Nova logs `Binding failed for port <port-id> and host <compute>.` and the migration
does not start.

### Why a driver refuses (the server log says which)

| Neutron server log | Driver | Meaning |
|---|---|---|
| `Refusing to bind port <port-id> to dead agent: ...` | ML2/OVS, ML2/OVN | The agent (OVS agent, or `ovn-controller` seen as an agent) on that host is not alive |
| `Refusing to bind port <port-id> due to no OVN chassis for host: <host>` | ML2/OVN | No `ovn-controller` registered under that host name |
| `Port <port-id> on network <network> not bound, no agent of type ... registered on host <host>` (debug level) | ML2/OVS | No OVS agent registered under that host name |
| `Refusing to bind port <port-id> on host <host> due to the OVN chassis bridge mapping physical networks [...] not supporting physical network: <physnet>` | ML2/OVN | The network is flat or VLAN on a physical network that host has no bridge mapping for |
| `Network <network> with segment <id> is connected to physical network <physnet>, but agent <host> reported physical networks [...]` (debug level) | ML2/OVS | Same, for the OVS agent's `bridge_mappings` |
| `Network <network> with segment <id> is type of <type> but agent <host> or mechanism driver only support [...]` (debug level) | ML2/OVS | The agent does not handle that network type (for example `vxlan` with tunnelling not set up on that host) |

The debug-level lines appear only with `debug = True` on the Neutron server; at the default level, the `Failed to
bind port` error is the only line, and the agent list and network type tell which case it is.

## Possible causes

### Cause 1: the agent on that host is dead

The OVS agent or `ovn-controller` on the compute is stopped, cannot reach the message bus (OVS) or the OVN southbound
database (OVN), or its clock drifted so that its reports arrive late. Quick check 3 shows it not alive.

### Cause 2: host names do not match

Nova binds the port to the compute's `host` (by default the system host name), and the agent registers under its own
`host`. If one uses a short name and the other the fully qualified name, or the host was renamed, Neutron finds no
agent for the name Nova gives. Kolla-Ansible 2026.1 sets neither for libvirt computes, so both use the host name the
system reports; a host renamed after deployment is the usual trigger.

### Cause 3: the host has no mapping for the network's physical network

A flat or VLAN network on a physical network (`physnet1`, ...) that this host's bridge mappings do not include: a
compute added without the provider network, or `neutron_physical_networks` / `bridge_mappings` set per host and
missing one ([provider networks](https://github.com/OzarLand/kolla-ansible-deployment-patterns/blob/main/networking/provider-networks-instances-on-flat-or-vlan-networks.md)).

### Cause 4: the network type is not handled on that host

A VXLAN or Geneve network on a host where the agent does not handle that type, or a `vnic_type` (`direct` for SR-IOV)
that no driver on that host supports.

## Resolution

### For cause 1: bring the agent back

Start the agent and fix what kept it from reporting (message bus, OVN southbound, clock: see
[compute clock skew](../diagnostics/compute-clock-skew.md)).

> ⚠️ **State-changing.** Restarts the agent on that compute; instances keep their network while it restarts
> ([restarting the OVS agent](restarting-the-ovs-agent-does-not-cut-instances.md)). Undo: none needed.

**Kolla-Ansible:** `sudo docker restart neutron_openvswitch_agent` (ML2/OVS) or `sudo docker restart ovn_controller`
(ML2/OVN).

**Any:** restart `neutron-openvswitch-agent` or `ovn-controller`.

With ML2/OVS, the Neutron server retries the binding of the host's `binding_failed` ports when the OVS agent comes back
alive (`_rebind_on_revive_agent_types` in `neutron/plugins/ml2/plugin.py`). For a failed build, create the instance
again; for a migration, run it again.

### For cause 2: make the names match

Compare `openstack compute service list --service nova-compute` with `openstack network agent list --host <compute>`.
Restore the host name the services were deployed with, or set the same `host` value for Nova and the agents on that
compute, then restart them. A renamed compute is otherwise a new compute for Nova and Neutron; renaming it is outside
this page.

### For cause 3: add the mapping, or keep instances of that network off the host

Add the physical network to that host's mappings (with Kolla-Ansible: `neutron_bridge_name`,
`neutron_external_interface` and `neutron_physical_networks` for the host, then a reconfigure of Neutron and, for OVN,
`ovn-controller`), and the switch configuration for it.

> ⚠️ **State-changing.** `kolla-ansible reconfigure -i <inventory> --tags neutron,ovn-controller --limit <compute>`
> changes the bridges and mappings on that compute and restarts its Neutron containers; instances keep their network
> if the existing mappings are unchanged. Undo: revert the variables and reconfigure again.

Or, if that host is not meant to carry the network, keep such instances off it (an aggregate and flavor or image
properties); the scheduler does not check physical networks by default.

### For cause 4: match the network type to the hosts

Use a network type the host's agent handles, or configure that type on the host. For `direct` ports, the host needs
SR-IOV and its agent; this page does not cover it.

## Verification

```bash
openstack network agent list --host <compute>          # alive, under the name Nova uses
openstack server create --image <image> --flavor <flavor> --network <network> \
  --availability-zone nova:<compute> --wait <test-name>
openstack port list --server <test-name> -c ID -c Status   # ACTIVE
openstack port show <port-id> -c binding_vif_type          # ovs (not binding_failed)
openstack server delete --wait <test-name>
```

> ⚠️ **State-changing.** Creates a test instance on that compute (`--availability-zone nova:<compute>` needs admin
> rights), deleted at the end.

## Validity and limits

- Written from Neutron 2026.1 (`neutron/plugins/ml2/managers.py` `_bind_port_level`,
  `neutron/plugins/ml2/drivers/mech_agent.py` `bind_port`, `check_segment_for_agent`,
  `neutron/plugins/ml2/drivers/ovn/mech_driver/mech_driver.py` `bind_port`, `neutron/plugins/ml2/plugin.py`), Nova
  2026.1 (`nova/network/neutron.py`, `nova/compute/manager.py`, `nova/exception.py`) and Kolla-Ansible 2026.1 (no
  `host` override for libvirt computes and Neutron agents). Not reproduced on a lab.
- Not covered: SR-IOV, hardware offload and smart NICs, routed provider networks, Ironic ports, and port binding for
  Octavia or other services' ports.

## Related

- [Instance stuck in BUILD, then ERROR after vif_plugging_timeout](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md):
  the port was bound, but the agent never reported it plugged.
- [New instance in ERROR with "Exceeded maximum number of retries"](../scheduling/build-fails-exceeded-maximum-number-of-retries.md)
- [Moving instances off a compute host](../migration/moving-instances-off-a-host-for-maintenance.md): live migration
  pre-checks.
- Kolla-Ansible: [provider networks](https://github.com/OzarLand/kolla-ansible-deployment-patterns/blob/main/networking/provider-networks-instances-on-flat-or-vlan-networks.md)
- Neutron 2026.1: [ML2 configuration](https://docs.openstack.org/neutron/2026.1/admin/config-ml2.html)

---

*Provenance: written by agent:ozarland-author, from the Neutron, Nova and Kolla-Ansible 2026.1 source and
documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=networking/port-binding-failed.md&title=Feedback%3A+networking/port-binding-failed.md):
your release, your deployment method, and what happened.
