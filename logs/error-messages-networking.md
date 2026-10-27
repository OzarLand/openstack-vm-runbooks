---
title: "Error messages: networking. Binding failed for port, Timeout waiting for network-vif-plugged"
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [nova, neutron, ovs, ovn]
deployment_methods: [any]
environment:
  topology: any
  network: ml2/ovs or ml2/ovn
  storage: any
  os: any
risk: read-only
confirmed_by: []
upstream_refs:
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/exception.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/network/neutron.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/compute/manager.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/virt/libvirt/driver.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/conf/compute.py
  - https://github.com/openstack/neutron/blob/stable/2026.1/neutron/plugins/ml2/managers.py
  - https://github.com/openstack/neutron/blob/stable/2026.1/neutron/plugins/ml2/plugin.py
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Error messages: networking. Binding failed for port, Timeout waiting for network-vif-plugged

This page belongs to the dictionary of frequent OpenStack error messages. For each message: where it appears, what it
means, the usual causes, the next log to read, and the runbook that fixes it. The messages are the format strings of
Nova and Neutron `stable/2026.1`, not captured lines; `<…>` marks a value.

The two messages fail at different steps of attaching a port to an instance on a compute:

1. **Binding.** Neutron chooses how to wire the port on that host (a mechanism driver binds it). If none can,
   the port's `binding:vif_type` becomes `binding_failed`, and Nova reports **Binding failed for port**.
2. **Plugging.** The port is bound, Nova creates the domain and plugs the interface, then waits for Neutron to send
   `network-vif-plugged` once the agent or `ovn-controller` has wired it. If the event does not come in time, Nova
   reports a **timeout**.

How to read the Neutron side in detail is on [reading the Neutron logs with ML2/OVS](reading-neutron-logs-ml2-ovs.md)
and [reading the OVN logs](reading-ovn-logs.md).

## `Binding failed for port <port-id>, please check neutron logs for more information.`

- **Where:** the message of `PortBindingFailed`
  ([`nova/exception.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/exception.py)), in
  `nova-compute.log` and often in the instance fault after a reschedule. Nova raises it when the port comes back with
  `binding:vif_type` `binding_failed` (`_ensure_no_port_binding_failure`), or when creating a port binding for a live
  migration fails, after logging `Binding failed for port <port-id> and host <host>.` (ERROR)
  ([`nova/network/neutron.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/network/neutron.py)).
- **Means:** Neutron could not bind the port on that host. Nova's message only points to Neutron; the reason is in
  `neutron-server.log`.
- **Read next:** `neutron-server.log` for the port id. From
  [`ml2/managers.py`](https://github.com/openstack/neutron/blob/stable/2026.1/neutron/plugins/ml2/managers.py):

  | Line (format) | Meaning |
  |---|---|
  | `Failed to bind port <port-id> on host <host> for vnic_type <vnic> using segments <segments>` (ERROR) | No mechanism driver could bind the port on that host |
  | `Failed to bind port <port-id> on host <host> at level <n>` (WARNING) | A hierarchical binding failed at one level |
  | `Failed to bind port <port-id> on host <host> allocated on resource providers: <providers>, because no mechanism driver reports being responsible` (ERROR) | The port has a resource request (for example QoS minimum bandwidth) and no driver claims the provider it was allocated on |

  `Attempt <n> to bind port <port-id>` (INFO, `ml2/plugin.py`) shows the server retrying the binding.
- **Usual causes:** no live agent of the right type on that host (agent down, or not deployed there); the agent's
  `bridge_mappings` do not include the network's physical network; a segment type the host does not support; the
  host name Nova sends differs from the agent's `host`; with OVN, the chassis is missing or lacks the bridge mapping.
- **Fix:** [Binding failed for port](../networking/port-binding-failed.md).

## `Timeout waiting for [('network-vif-plugged', '<port-id>')] for instance with vm_state <state> and task_state <state>. Event states are: <states>`

- **Where:** WARNING in `nova-compute.log`, from `wait_for_instance_event` in
  [`compute/manager.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/compute/manager.py). The list holds
  the events still expected. With `vif_plugging_is_fatal` true (the default), the libvirt driver then powers the guest
  off, cleans up and fails the build with `VirtualInterfaceCreateException`, whose message is `Virtual Interface
  creation failed` ([`virt/libvirt/driver.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/virt/libvirt/driver.py)).
  On the lab run of the runbook linked below, the instance fault then read `Build of instance … aborted: Failed to
  allocate the network(s), not rescheduling`: this failure is not rescheduled to another compute.
  The wait lasts `vif_plugging_timeout`, 300 s by default
  ([`conf/compute.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/conf/compute.py)).
- **Means:** the port is bound, but Neutron never told Nova that it was wired on the host within the timeout.
- **Related line:** `Neutron Reported failure on event <event> for instance <uuid>` (ERROR, same driver file): Neutron
  sent the event with a failure status instead of never sending it; with `vif_plugging_is_fatal` it also ends in
  `VirtualInterfaceCreateException`.
- **Read next, in order, by port id:**
  1. On the compute: did the agent (ML2/OVS, `neutron-openvswitch-agent.log`) or `ovn-controller` (`Claiming lport
     <port-id> for this chassis.`) wire the port?
  2. On the controllers: did `neutron-server` mark the port up and send the event to Nova (`Nova event matching …
     response: …`)?
  3. In `nova-api.log`: was the event received (`Creating event network-vif-plugged:<port-id> …`)?

  The step after the last line you find is where the chain stopped.
- **Usual causes:** the agent on the compute is down or slow (RPC timeouts, a large backlog after a restart);
  `ovs-vswitchd` or `ovn-controller` not working on that compute; Neutron cannot reach Nova's API to send the event
  (credentials or endpoint in `[nova]` of Neutron's configuration); RabbitMQ problems.
- **Fix:** [Instance stuck in BUILD, waiting for vif_plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md).

## A short reading order

1. `openstack port show <port-id> -c binding_vif_type -c binding_host_id -c status`: `binding_failed` → binding;
   bound with status `DOWN` → plugging.
2. Binding: `neutron-server.log` for `Failed to bind port <port-id>`, then the agents of that host.
3. Plugging: follow the port id from the compute's agent or `ovn-controller` to `neutron-server` to `nova-api`.

## Validity and limits

- Written from Nova and Neutron `stable/2026.1` source only. Every message is a format string from that source, not a
  captured line. The list of events in the timeout line is a list of `(name, tag)` tuples (`_get_neutron_events` in
  the libvirt driver), which matches the line recorded on the lab in the vif_plugged runbook.
- Not run on a lab yet. The two runbooks linked above have their own observations; captures of both messages with
  ML2/OVS and ML2/OVN are requested.
- SR-IOV and other vnic types, and hierarchical port binding, are only mentioned.

---

*Provenance: written by agent:ozarland-author, from upstream source code (Nova, Neutron). Reviewed in its pull request
(see the page history). Checked by a human operator: not yet.*
