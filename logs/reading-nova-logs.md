---
title: Reading the Nova logs. What normal looks like, and what the common lines mean
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [nova, placement, rabbitmq, libvirt]
deployment_methods: [kolla-ansible, any]
environment:
  topology: multinode
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by: []
upstream_refs:
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/compute/manager.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/compute/resource_tracker.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/conductor/manager.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/conductor/api.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/scheduler/manager.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/filters.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/servicegroup/drivers/db.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/virt/libvirt/host.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/virt/libvirt/driver.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/api/openstack/wsgi.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/exception.py
  - https://opendev.org/openstack/oslo.messaging/src/branch/stable/2026.1/oslo_messaging/_drivers/amqpdriver.py
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Reading the Nova logs. What normal looks like, and what the common lines mean

Nova writes four logs that matter for an instance: `nova-api`, `nova-conductor`, `nova-scheduler` (on the
controllers) and `nova-compute` (on each compute). Each answers a different question. This page says what each
one logs when things are fine, which lines signal a problem, what they mean, and where to look next. Every message
quoted here is the message text from Nova's 2026.1 source (file named in each table); the values in angle brackets
are placeholders.

File locations and the line format: [where the logs are](where-openstack-logs-are-and-how-to-read-a-line.md).
Following one request through these logs: [what a VM creation traverses](../diagnostics/what-a-vm-creation-traverses-and-tracing-by-request-id.md).

## Which log answers which question

| Question | Log | Where |
|---|---|---|
| Did the API accept the request, and what did it answer? | `nova-api.log`, and the uWSGI request line in `nova-api-uwsgi.log` on Kolla-Ansible | controllers |
| Was a host found? Why not? | `nova-scheduler.log` | controllers |
| What happened between the API and the compute, and after a failure (retries, cell0)? | `nova-conductor.log` | controllers |
| What did the compute do with the instance: claim, network, disk, hypervisor? | `nova-compute.log` | the compute in `OS-EXT-SRV-ATTR:host`, or every compute that tried |

Nova's own record of what happened to an instance is often faster than the logs, and tells you which compute to
read:

```bash
openstack server show <server-id> -c status -c fault -c OS-EXT-SRV-ATTR:host
openstack server event list <server-id>
openstack server event show <server-id> <request-id>      # per-step result; tracebacks for admins
```

## When a service starts

Every Nova service built on `nova.service` logs, at start:

```text
Starting <topic> node (version <version>)
```

(`nova/service.py`, `INFO`; `<topic>` is `compute`, `conductor` or `scheduler`). Use it to find restarts:

```bash
grep -n "Starting compute node" /var/log/kolla/nova/nova-compute.log | tail -5
```

A service that started before `nova-conductor` was reachable logs, until the conductor answers
(`nova/conductor/api.py`):

| Line | Level | Meaning |
|---|---|---|
| `Timed out waiting for nova-conductor.  Is it running? Or did this service start before nova-conductor?  Reattempting establishment of nova-conductor connection...` | WARNING | The service sent an RPC ping to the conductor and got no reply. It retries; it does not stop. Repeated: the conductor is down, or RabbitMQ does not deliver. |
| `nova-conductor connection established successfully` | INFO | The wait is over. Logged only after at least one timeout. |

## nova-compute

### What normal looks like

- **At start**, the libvirt driver logs the host's capabilities (`Libvirt host capabilities <xml>`, `INFO`,
  `nova/virt/libvirt/host.py`), and on the first start of a new compute the resource tracker logs
  `Compute node record created for <host>:<node> with uuid: <uuid>` (`INFO`, `resource_tracker.py`).
- **Every periodic run** of the resource tracker (every 60 s by default), at `DEBUG` only:
  `Final resource view: name=<node> phys_ram=<n>MB used_ram=<n>MB phys_disk=<n>GB used_disk=<n>GB total_vcpus=<n> used_vcpus=<n> pci_stats=[...] stats={...}`.
  With debug on, this is the compute's own view of its capacity, to compare with Placement.
- **For each instance** it builds: `Claim successful on node <node>` (`claims.py`, `INFO`), then
  `Took <n> seconds to spawn the instance on the hypervisor.` (`INFO`), then libvirt lifecycle events as
  `VM <state> (Lifecycle Event)` (`INFO`; for example `Started`, `Resumed`, `Paused`, `Stopped`), all with
  `[instance: <uuid>]`.

At the default `INFO` level, a healthy compute is quiet between builds. A periodic `[-]` line repeating every minute
at `WARNING` or above is worth reading.

### Lines that signal a problem

| Line (message text) | Level | What it means | Next |
|---|---|---|---|
| `Failed to build and run instance` followed by a traceback | ERROR | The build failed on this compute for a reason not handled specifically. The traceback's last line is the cause. Nova may retry on another compute: the fault then reads `Build of instance <uuid> was re-scheduled: <reason>` here, and the next compute logs its own attempt. | [Exceeded maximum number of retries](../scheduling/build-fails-exceeded-maximum-number-of-retries.md) |
| `Instance failed to spawn` followed by a traceback | ERROR | The failure happened inside the build's resource setup or the hypervisor spawn. Read the traceback: a libvirt error, a disk or image problem, a timeout. | the traceback's last line; libvirt's log on the compute |
| `Failed to allocate network(s)` with a traceback, then the fault `Build of instance <uuid> aborted: Failed to allocate the network(s), not rescheduling.` | ERROR | The network part of the build failed, often the wait for `network-vif-plugged`. Aborted, **not retried** on another compute. | [Waiting for vif_plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md) |
| `Timeout waiting for [('network-vif-plugged', '<port-id>')] for instance with vm_state <state> and task_state <state>. Event states are: <...>` | WARNING | Nova waited `vif_plugging_timeout` seconds for Neutron's event and none came. The lab-verified runbook shows this line in full. | [Waiting for vif_plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md); follow the port id in `neutron-server.log` and the agent log of the compute |
| `Build of instance <uuid> aborted: Failure prepping block device.` | ERROR (fault) | The boot volume or block device mapping could not be prepared. Not retried. | [Boot from volume fails](../scheduling/instance-fails-to-boot-from-volume.md) |
| `Build of instance <uuid> aborted: <reason>` with a flavor or image reason (for example `FlavorDiskTooSmall`, `ImageNotActive`) | ERROR (fault) | The request cannot work on any compute; Nova does not retry it. | fix the flavor or image |
| `Insufficient compute resources: <reason>.` | (exception text) | The claim failed on this compute: its own view of capacity disagreed with the scheduler's. Retried on an alternate. | Placement and the resource tracker; [No valid host](../scheduling/no-valid-host-with-free-capacity.md) |
| `Connection to libvirt lost: <reason>`, then `Connection event '0' reason 'Connection to libvirt lost: <reason>'` | INFO | nova-compute lost its connection to libvirtd. The libvirt driver **disables the compute service** with a reason starting with `AUTO: ` (`_set_host_enabled` in `driver.py`), and re-enables it when the connection comes back. | `openstack compute service list --long` (Disabled Reason); libvirt's log; [restarting nova_libvirt](../lifecycle/restarting-nova-libvirt-does-not-stop-instances.md) |
| `Connection to libvirt failed: <error>` with a traceback | ERROR | nova-compute could not open a connection to libvirtd at all. | Is libvirtd (`nova_libvirt` on Kolla-Ansible) running? |
| `Error updating resources for node <node>.` with a traceback | ERROR | The periodic resource update failed: Placement unreachable, a database or RPC error, or a driver error. The compute's inventory in Placement is not refreshed while this lasts. | the traceback; Placement's log |
| `Lost connection to nova-conductor for reporting service status.` | WARNING | The service heartbeat (sent through the conductor over RabbitMQ) timed out. Logged once; `Recovered from being unable to report status.` (`INFO`) when it works again. Long enough, and the service shows `down`. | RabbitMQ; nova-conductor; [nova-compute down](../recovery/nova-compute-down-instances-still-running.md) |
| `Instance shutdown by itself. Calling the stop API. Current vm_state: <state>, current task_state: <state>, original DB power_state: <n>, current VM power_state: <n>` | WARNING | The power state sync found the domain shut down while Nova thought it ran. Nova records the stop. The cause is inside the guest or on the host (OOM killer, QEMU crash). | QEMU's log of the instance; the host's kernel log |

Sources: `nova/compute/manager.py`, `nova/compute/claims.py`, `nova/compute/resource_tracker.py`,
`nova/servicegroup/drivers/db.py`, `nova/virt/libvirt/host.py`, `nova/virt/libvirt/driver.py`,
`nova/exception.py` (`BuildAbortException`, `RescheduledException`, `ComputeResourcesUnavailable`).

**Kolla-Ansible:** on the compute:

```bash
grep -nE ' (ERROR|CRITICAL) ' /var/log/kolla/nova/nova-compute.log | tail -20
grep -n "<server-id>" /var/log/kolla/nova/nova-compute.log
grep -nE "Connection to libvirt|Lost connection to nova-conductor|Error updating resources" /var/log/kolla/nova/nova-compute.log | tail
```

**Any:** the same `grep` on `/var/log/nova/nova-compute.log`, or `journalctl -u <unit>` piped to `grep`.

## nova-scheduler

### What normal looks like

At `INFO`, the scheduler logs little for a successful request. With debug on, it logs each filter's result and the
weighing; that is the level to use when you need to know why a host was not chosen.

### Lines that signal a problem

| Line (message text) | Level | What it means | Next |
|---|---|---|---|
| `Got no allocation candidates from the Placement API. This could be due to insufficient resources or a temporary occurrence as compute nodes start up.` | INFO | Placement returned no resource provider that has the requested resources and traits. The scheduler raises `NoValidHost` without running any filter. | Placement: inventories, allocations, traits; [No valid host](../scheduling/no-valid-host-with-free-capacity.md) |
| `Filtering removed all hosts for the request with instance ID '<uuid>'. Filter results: [...]` | INFO (short form), DEBUG (full form) | Placement found candidates, then the filters removed every one. The `Filter results` list shows how many hosts each filter kept, in order: the filter after which the count drops to 0 is the one to look at. | the filter's configuration; [No valid host](../scheduling/no-valid-host-with-free-capacity.md) |

Sources: `nova/scheduler/manager.py`, `nova/filters.py`.

```bash
grep -n "Got no allocation candidates\|Filtering removed all hosts" /var/log/kolla/nova/nova-scheduler.log | tail
```

A request that fails here never reaches a compute: there is nothing in any `nova-compute.log` for it.

## nova-conductor

### Lines that signal a problem

| Line (message text) | Level | What it means | Next |
|---|---|---|---|
| `Failed to schedule instances` with a traceback ending in `NoValidHost: No valid host was found. <reason>` | ERROR | The scheduler found no host. The conductor puts the instance in `ERROR` in **cell0** (the instance never reaches a cell database), so `server show` works but there is nothing on any compute. | the scheduler's log for the same request id |
| `Exceeded maximum number of retries. Exhausted all hosts available for retrying build failures for instance <uuid>.` | (fault, `MaxRetriesExceeded`) | Computes were chosen and each failed; no alternate was left. | each compute that tried: [max retries runbook](../scheduling/build-fails-exceeded-maximum-number-of-retries.md) |
| `Host '<host>' is not mapped to any cell` | (exception text) | A compute exists that no cell knows about, usually a new compute before host discovery. | `nova-manage cell_v2 list_hosts`; [VM creation traverses](../diagnostics/what-a-vm-creation-traverses-and-tracing-by-request-id.md) |

Sources: `nova/conductor/manager.py`, `nova/exception.py` (`NoValidHost`, `MaxRetriesExceeded`,
`HostMappingNotFound`).

## nova-api

### What normal looks like

The application log (`nova-api.log`) records errors and some actions; on Kolla-Ansible 2026.1 the per-request line
(method, path, status, duration) is in uWSGI's log, `nova-api-uwsgi.log`.

### Lines that signal a problem

| Line (message text) | Level | What it means | Next |
|---|---|---|---|
| `HTTP exception thrown: <message>` | INFO | The API answered a 4xx on purpose (not found, conflict, bad request). Usually a client error, not a platform one. | the message itself |
| `Unexpected exception in API method` with a traceback; the client gets `Unexpected API Error.` and HTTP 500 | ERROR | An unhandled exception inside the API: often a dependency (database, RabbitMQ, Keystone, Placement) failing underneath. | the traceback; [shared dependencies](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md) |
| `MessagingTimeout: Timed out waiting for a reply to message ID <msg-id>` in a traceback | ERROR | A synchronous RPC call (to the conductor, or to a compute) got no reply within `rpc_response_timeout` (60 s by default in `oslo.messaging`). The other side is down, overloaded, or RabbitMQ did not deliver. | RabbitMQ; the service that should have answered |

Sources: `nova/api/openstack/wsgi.py`; `oslo_messaging/_drivers/amqpdriver.py` for the `MessagingTimeout` text.

## Reading a failure in order

1. **`server show` fault and `server event list`**: which action, which request id, and whether a host was assigned.
2. **No host assigned:** read `nova-scheduler.log` and `nova-conductor.log` for the request id. Placement or a
   filter decided.
3. **A host assigned:** read that compute's `nova-compute.log` for the instance UUID. With retries, read every compute
   listed in `server event show`.
4. **The last line of the traceback** names the exception; the lines just above it name the call that raised it.
5. **Dependencies:** `MessagingTimeout`, database errors and 401s in several services at once point below Nova.

## Validity and limits

- Every message on this page is the message text from Nova 2026.1 (and `oslo.messaging` 2026.1 for
  `MessagingTimeout`), read in the source files listed above. **None was captured on a lab for this page**, except
  the `Timeout waiting for ...` and `Build of instance ... aborted` lines, which the lab-verified
  [vif_plugged runbook](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md) shows. A lab request asks
  for real samples of the others.
- Level and exact wording can change between releases; a message text quoted here is for 2026.1.
- Only the libvirt driver is covered.
- "What normal looks like" lists the lines that the source logs at `INFO` or `DEBUG`; how many of them appear on
  your platform depends on its `debug` setting and its `default_log_levels`.

## Related

- [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md)
- [What a VM creation traverses](../diagnostics/what-a-vm-creation-traverses-and-tracing-by-request-id.md)
- [No valid host with free capacity](../scheduling/no-valid-host-with-free-capacity.md)
- [Exceeded maximum number of retries](../scheduling/build-fails-exceeded-maximum-number-of-retries.md)
- [Waiting for vif_plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md)
- [Logs index](README.md)

---

*Provenance: written by agent:ozarland-author, from Nova 2026.1 source code and documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Freading-nova-logs.md&title=Feedback%3A+logs%2Freading-nova-logs.md):
your release, your deployment method, and what happened.
