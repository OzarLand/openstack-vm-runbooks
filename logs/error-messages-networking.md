---
title: "Error messages: networking. Binding failed for port, Timeout waiting for network-vif-plugged, port creation refusals"
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
  - https://github.com/openstack/neutron/blob/stable/2026.1/neutron/notifiers/nova.py
  - https://github.com/openstack/neutron/blob/stable/2026.1/neutron/ipam/exceptions.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/conductor/tasks/live_migrate.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/api/openstack/compute/server_external_events.py
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Error messages: networking. Binding failed for port, Timeout waiting for network-vif-plugged, port creation refusals

This page belongs to the dictionary of frequent OpenStack error messages. For each message: where it appears, what it
means, the usual causes, the next log to read, and the runbook that fixes it. The messages are the format strings of
Nova and Neutron `stable/2026.1`, not captured lines; `<…>` marks a value.

The two messages fail at different steps of attaching a port to an instance on a compute:

1. **Binding.** Neutron chooses how to wire the port on that host (a mechanism driver binds it). If none can,
   the port's `binding:vif_type` becomes `binding_failed`, and Nova reports **Binding failed for port**.
2. **Plugging.** The port is bound, Nova creates the domain and plugs the interface, then waits for Neutron to send
   `network-vif-plugged` once the agent or `ovn-controller` has wired it. If the event does not come in time, Nova
   reports a **timeout**.

Before both, Neutron can refuse to create the port at all (no free address, quota, MAC or address in use), and the
API can refuse the request before any compute is involved; those are in the last sections.

How to read the Neutron side in detail is on [reading the Neutron logs with ML2/OVS](reading-neutron-logs-ml2-ovs.md)
and [reading the OVN logs](reading-ovn-logs.md).

## `Binding failed for port <port-id>, please check neutron logs for more information.`

- **Where:** the message of `PortBindingFailed`
  ([`nova/exception.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/exception.py)), in
  `nova-compute.log` and often in the instance fault after a reschedule. Nova raises it when the port comes back with
  `binding:vif_type` `binding_failed` (`_ensure_no_port_binding_failure`), or when creating a port binding for a live
  migration fails, after logging `Binding failed for port <port-id> and host <host>.` (ERROR)
  ([`nova/network/neutron.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/network/neutron.py)).
  - **Build:** the failure surfaces inside the `Instance failed to spawn` and `Failed to build and run instance`
    tracebacks (ERROR), then `Build of instance <uuid> was re-scheduled: Binding failed for port …`: the build goes
    back to the scheduler, and when every host fails the same way the fault is `Exceeded maximum number of retries`
    ([runbook](../scheduling/build-fails-exceeded-maximum-number-of-retries.md)).
  - **Live migration:** in `nova-conductor.log`, which binds the ports on the destination before the migration
    starts (`_bind_ports_on_destination` in `conductor/tasks/live_migrate.py`); the error becomes `Migration
    pre-check error: Binding failed for port …`, and the conductor may try another host.
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
  2. On the controllers: did `neutron-server` mark the port up and send the event to Nova? Neutron sends
     `network-vif-plugged` when the port status goes from `DOWN`, `BUILD` or unset to `ACTIVE` (event status
     `completed`) or to `ERROR` (status `failed`, which gives the `Neutron Reported failure` line above), and logs
     Nova's answer as `Nova event matching [<request-ids>] response: {…}` (INFO; `Nova event response: …` without
     request ids), `… returned with failed status` (WARNING), or `Failed to notify nova on events: …` (ERROR, with
     the traceback) ([`neutron/notifiers/nova.py`](https://github.com/openstack/neutron/blob/stable/2026.1/neutron/notifiers/nova.py)).
  3. In `nova-api.log`: was the event received (`Creating event network-vif-plugged:<port-id> …`)?

  The step after the last line you find is where the chain stopped.
- **Usual causes:** the agent on the compute is down or slow (RPC timeouts, a large backlog after a restart);
  `ovs-vswitchd` or `ovn-controller` not working on that compute; Neutron cannot reach Nova's API to send the event
  (credentials or endpoint in `[nova]` of Neutron's configuration); RabbitMQ problems.
- **Fix:** [Instance stuck in BUILD, waiting for vif_plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md).

## Port creation errors: no more fixed IPs, IP already in use, port quota, MAC in use

When Nova creates the instance's port (`--network`), Neutron can refuse. Nova logs a WARNING starting with `Neutron
error:` (`_create_port_minimal` and `_update_port` in `nova/network/neutron.py`) and turns the refusal into its own
exception:

| Nova's WARNING in `nova-compute.log` | Nova exception (fault text) | Neutron's refusal | Next |
|---|---|---|---|
| `Neutron error: No more fixed IPs in network: <network-id>` | `NoMoreFixedIps`: `No fixed IP addresses available for network: <network-id>`; the build is aborted, not rescheduled: `Build of instance <uuid> aborted: Failed to allocate the network(s) with error No fixed IP addresses available for network: <network-id>, not rescheduling.` | `No more IP addresses available for subnet <subnet-id>.` (`IpAddressGenerationFailure`, 409) | the subnet's allocation pools and the ports using them: `openstack port list --network <network-id>`; orphan ports: [stale records, orphan ports](../lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md) |
| `Neutron error: Fixed IP <ip> is already in use.` | `FixedIpAlreadyInUse`: `Fixed IP <ip> is already in use.` | `IP address <ip> already allocated in subnet <subnet-id>` (`IpAddressAlreadyAllocated`, 409), or `IpAddressInUse` | the port that holds the address: `openstack port list --fixed-ip ip-address=<ip>` |
| `Neutron error: <ip> is not a valid IP address for network <network-id>.` | `InvalidInput`: `Fixed IP <ip> is not a valid ip address for network <network-id>.` | `InvalidIpForNetwork`: the address is not in any subnet of that network | the subnets' CIDRs |
| `Neutron error: Port quota exceeded in tenant: <project-id>` | `PortLimitExceeded`: `Maximum number of ports exceeded` | Neutron's quota refusal (`OverQuota`) | `openstack quota show <project-id>` and the project's ports |
| `Neutron error: MAC address <mac> is already in use on network <network-id>.` | `PortInUse`: `Port <mac> is still in use.` (Nova puts the MAC where the port id usually goes) | `MacAddressInUse` | the port that has that MAC: `openstack port list --mac-address <mac>` |
| `Neutron error: Tried to bind a port with fixed_ips to a host in the wrong segment on network <network-id>.` | `FixedIpInvalidOnHost` | a routed provider network: the port's address is on a segment the host is not attached to | the network's segments and the host's segment |

The port quota is also checked by `nova-api` before the build starts: when the project already has more ports than
its quota, the API refuses with `The number of defined ports: <n> is over the limit: <n>` (`PortLimitExceeded`); the
instance is not created.

Any other Neutron error while creating the port is logged as `Neutron error creating port on network <network-id>`
(ERROR, with the traceback), followed by the Neutron client's message, which names the cause.

## Live migration: binding activation and clean-up

During a live migration Nova keeps two bindings per port: the active one on the source, and an inactive one on the
destination that it activates when the guest switches over.

| Line | Where | Means | Next |
|---|---|---|---|
| `Unexpected error trying to activate binding for port <port-id> and host <host>.` (ERROR) then `Failed to activate binding for port <port-id> on host <host>; please check neutron logs for more information` (`PortBindingActivationFailed`) | `nova-compute.log` on the source, after the guest has moved | Neutron refused to make the destination binding active. The guest runs on the destination but its port may still be wired for the source | `neutron-server.log` for the port id; the migration and the port's `binding_host_id` |
| `Binding for port <port-id> and host <host> is already active` (WARNING) | `nova-compute.log` on the source | Neutron answered 409: the binding was already active. Nova continues | none |
| `Failed to delete binding for port(s) <port-ids> on host <host>; please check neutron logs for more information` (`PortBindingDeletionFailed`), logged as `Failed to delete port bindings from source host. Error: …` (ERROR) | `nova-compute.log` on the source | Nova could not remove the old binding on the source host. Nova goes on with the clean-up; the port keeps an extra inactive binding | the port's bindings in Neutron |

The binding refusal before a live migration starts is the first entry of this page (`Migration pre-check error:
Binding failed for port …`).

## Refusals at the API, before any log on a compute

These come back to the user at once (HTTP 400 or 403 from `nova-api`), and the instance is not built. They name the
problem in the message itself:

| Message | Exception |
|---|---|
| `Network <network-id> requires a subnet in order to boot instances on.` | `NetworkRequiresSubnet` |
| `More than one possible network found. Specify network ID(s) to select which one(s) to connect to.` | `NetworkAmbiguous` |
| `It is not allowed to create an interface on external network <network-id>` | `ExternalNetworkAttachForbidden` |
| `Port <port-id> is still in use.` | `PortInUse` (a `--port` already bound to another instance) |
| `Port <port-id> requires a FixedIP in order to be used.` | `PortRequiresFixedIP` |
| `Port <port-id> not usable for instance <uuid>.` | `PortNotUsable` |
| `Network <network-id> could not be found.` / `Port id <port-id> could not be found.` | `NetworkNotFound` / `PortNotFound` |

## A short reading order

1. `openstack port show <port-id> -c binding_vif_type -c binding_host_id -c status`: `binding_failed` → binding;
   bound with status `DOWN` → plugging.
2. Binding: `neutron-server.log` for `Failed to bind port <port-id>`, then the agents of that host.
3. Plugging: follow the port id from the compute's agent or `ovn-controller` to `neutron-server` to `nova-api`.
4. `No fixed IP addresses`, `already in use`, `ports exceeded`: the network, subnet and project quota in Neutron.

## Validity and limits

- Written from Nova and Neutron `stable/2026.1` source only. Every message is a format string from that source, not a
  captured line. The list of events in the timeout line is a list of `(name, tag)` tuples (`_get_neutron_events` in
  the libvirt driver), which matches the line recorded on the lab in the vif_plugged runbook.
- Not run on a lab yet. The two runbooks linked above have their own observations; captures of both messages with
  ML2/OVS and ML2/OVN are requested.
- SR-IOV and other vnic types, and hierarchical port binding, are only mentioned.
- The vif-plugged wait is that of the libvirt driver. It also runs on a hard reboot, a resume and the revert of a
  resize (`_create_guest_with_network`), not on the destination of a cold migration (`vifs_already_plugged=True` in
  `finish_migration`).

---

*Provenance: written by agent:ozarland-author, from upstream source code (Nova, Neutron). Reviewed in its pull request
(see the page history). Checked by a human operator: not yet.*
