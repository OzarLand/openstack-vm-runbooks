---
title: A compute shows disabled or down. Why, what still works on it, and how to bring it back
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, placement, libvirt]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any multinode
  network: any
  storage: any
  os: any
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, multinode (3 controllers, 2 computes)", date: 2026-10-27, note: "the four causes produced on a disposable lab (nova_libvirt stopped, operator disable, nova_compute stopped, --down with an evacuation left done); quick checks, every resolution and Verification run; --enable while libvirt down, AUTO over an operator reason, the trait error, volume detach (no Cinder), the --enable after --up and forcing down a fenced host not run" }
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/admin/evacuate.html
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#DEFAULT.service_down_time
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#DEFAULT.running_deleted_instance_action
  - https://docs.openstack.org/api-ref/compute/#update-compute-service
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# A compute shows disabled or down. Why, what still works on it, and how to bring it back

`openstack compute service list` gives each `nova-compute` two independent fields. **Status** is
`enabled` or `disabled`: whether the scheduler may place new instances there. **State** is `up` or `down`: whether
Nova believes the service is alive, from its heartbeat or because an operator *forced* it down. Each combination
means something different for the instances already there, and each is set and cleared differently. Mixing them up
is how instances get lost: a compute forced down while it still runs its instances lets Nova evacuate or
locally delete them.

## Symptom

```text
$ openstack compute service list --service nova-compute --long
+------+--------------+--------+------+----------+-------+----------------------------+-----------------------------------------------+-------------+
| ID   | Binary       | Host   | Zone | Status   | State | Updated At                 | Disabled Reason                               | Forced Down |
+------+--------------+--------+------+----------+-------+----------------------------+-----------------------------------------------+-------------+
| ...  | nova-compute | <host> | nova | disabled | up    | 2026-10-26T10:00:05.000000 | AUTO: Connection to libvirt lost: <reason>    | False       |
| ...  | nova-compute | <host> | nova | disabled | down  | 2026-10-26T09:12:44.000000 | <operator reason>                             | True        |
| ...  | nova-compute | <host> | nova | enabled  | down  | 2026-10-26T08:58:31.000000 | None                                          | False       |
+------+--------------+--------+------+----------+-------+----------------------------+-----------------------------------------------+-------------+
```

`Forced Down` appears with compute API 2.11 or later, which recent clients negotiate.

What the operator notices first is usually one of:

- builds never land on that compute. The scheduler excludes it through the `COMPUTE_STATUS_DISABLED` trait before
  the filters run; the ComputeFilter debug line `<host-state> is disabled, reason: <reason>` shows only when the
  trait is missing or out of sync;
- the scheduler log has, at warning level, `<host-state> has not been heard from in a while` (ComputeFilter, from
  `nova/scheduler/filters/compute_filter.py`);
- a resize, cold migration or volume detach of one of its instances fails with `Service is unavailable at this time.`
  (HTTP 409), and other actions (reboot, stop) are accepted but never done, leaving a task state set;
- `openstack server evacuate` is refused with `Compute service of <host> is still in use.`, because the service is
  still `up`.

## Impact

| Status / State | New instances | Actions on its instances | Delete of its instances | Evacuate |
|---|---|---|---|---|
| enabled / up | yes | yes | normal | refused |
| disabled / up | no | yes | normal | refused |
| any / down, heartbeat lost | no | resize, cold migration, volume detach refused (`Service is unavailable at this time.`); others sent to a service that does not process them | **local delete**: the database record goes, the guest on the hypervisor is not touched | allowed |
| any / down, forced down | no | same refusals; if `nova-compute` still runs, it processes the others | **local delete** | allowed |

Running instances are not stopped by any of these states. The danger is in what Nova allows once the service is
`down`: a local delete leaves the guest running on the hypervisor, with its disk and its network, while Nova
forgets it; an evacuate starts a second copy of an instance that may still run.

## Quick checks

Read-only.

```bash
# 1. Status, state, reason, forced down, and the last heartbeat
openstack compute service list --service nova-compute --host <host> --long

# 2. Is nova-compute alive on the compute? (Kolla-Ansible)
sudo docker ps --filter name=nova_compute --filter name=nova_libvirt --format '{{.Names}} {{.Status}}'
sudo tail -n 50 /var/log/kolla/nova/nova-compute.log

# 3. Is the disabled trait set on its resource provider? Nova excludes it from scheduling through this trait.
#    <node-uuid> is the provider's UUID: openstack resource provider list --name <hypervisor-hostname>
openstack resource provider trait list <node-uuid> | grep COMPUTE_STATUS_DISABLED

# 4. Did libvirt drop? The libvirt driver logs the event it reacts to
sudo grep -e "Connection event" -e "Connection to libvirt lost" /var/log/kolla/nova/nova-compute.log | tail -5

# 5. Instances on it, and evacuations that left from it
openstack server list --all-projects --host <host> -c ID -c Status
openstack server migration list --host <host> --type evacuation
```

## Diagnosis

### Status: who disabled it, from the reason

- A reason starting with `AUTO: ` was set by the libvirt driver itself (`DISABLE_PREFIX` in
  `nova/virt/libvirt/driver.py`). It disables the service when its connection to libvirt drops
  (`Connection to libvirt lost: <reason>`, or just `Connection to libvirt lost`, from `nova/virt/libvirt/host.py`), and enables it again by itself when the
  connection comes back. It never overrides a reason that does not start with `AUTO: `.
- Any other reason, or none, was set through the API: an operator, a script, a maintenance tool.
- Disabling also adds the `COMPUTE_STATUS_DISABLED` trait to the compute's resource provider, so the scheduler's
  request filter (`compute_status_filter`) excludes it before the filters run. If the trait cannot be updated,
  `nova-compute` logs `An error occurred while updating COMPUTE_STATUS_DISABLED trait on compute node resource
  provider <uuid>`, and the periodic task fixes it later.

### State: heartbeat or forced

- `down` with `Forced Down` `False`: the last heartbeat (`Updated At`) is older than `[DEFAULT] service_down_time`
  (60 s by default; `nova-compute` reports every `report_interval`, 10 s). `nova-compute` is stopped, hung, cannot
  reach the database through the conductor, or the compute is gone. See
  [nova-compute down, instances still running](../recovery/nova-compute-down-instances-still-running.md).
- `down` with `Forced Down` `True`: someone ran `openstack compute service set --down`. Nova treats the service as
  down whatever its heartbeat says (`nova/servicegroup/api.py`), even if `nova-compute` runs fine. This is meant for
  a host that has been fenced, so that evacuation can start without waiting for `service_down_time`.

### What happens to an instance deleted while its compute is down

`nova/compute/api.py` does a *local delete* when the instance's compute service is not up: it removes the instance
from the database, its ports and volume attachments, and its allocations, but nothing on the compute. When
`nova-compute` runs again there, its periodic task `_cleanup_running_deleted_instances` finds the guest it no longer
knows and applies `[DEFAULT] running_deleted_instance_action` (default `reap`): `Destroying instance with name label
'<instance-name>' which is marked as DELETED but still present on host.` The task runs at once when `nova-compute`
starts (`run_immediately=True`), then every `running_deleted_instance_poll_interval` (1800 s by default): a
returning host destroys such a guest within about a minute of starting. While `nova-compute` stays down, the guest
runs, without Nova knowing it.

## Possible causes

| What you see | Cause |
|---|---|
| `disabled`, reason `AUTO: Connection to libvirt lost` or `AUTO: Connection to libvirt lost: ...`, state `up` | Cause 1: libvirt dropped; the service re-enables itself when it comes back |
| `disabled`, another reason, state `up` | Cause 2: disabled on purpose (maintenance, fencing) and never re-enabled |
| `down`, `Forced Down` `False` | Cause 3: no heartbeat |
| `down`, `Forced Down` `True` | Cause 4: forced down, usually after a host failure, and never set back up |

## Resolution

### For cause 1: let libvirt come back

Fix `nova_libvirt` (see [restarting nova_libvirt](../lifecycle/restarting-nova-libvirt-does-not-stop-instances.md)).
When the connection is back, the driver logs a connection event and re-enables the service; nothing to do by hand.
Do not run `--enable` while libvirt is still down: the service would be scheduled to and fail every build.

### For cause 2: enable it when the reason no longer holds

Read the reason first, and ask whoever set it.

> ⚠️ **State-changing.** Makes the compute schedulable again: new instances can land on it at once. Precondition:
> the reason for disabling is resolved, the service is `up`. Undo: `openstack compute service set --disable
> --disable-reason "<why>" <host> nova-compute`.

```bash
openstack compute service set --enable <host> nova-compute
```

### For cause 3: get the heartbeat back

Start or repair `nova-compute` on the compute
([nova-compute down, instances still running](../recovery/nova-compute-down-instances-still-running.md)). Until it
reports again, do not delete its instances: each delete would be a local delete.

### For cause 4: set it back up only after its evacuations are cleaned up

Nova refuses `--up` while evacuations from that host are still `done` (`nova/api/openstack/compute/services.py`):

```text
Unable to force up host <host> as `done` evacuation migration records remain associated with the host. Ensure the compute service has been restarted, allowing these records to move to `completed` before retrying this request.
```
`nova-compute` moves them to `completed` when it starts on the repaired host, after destroying the local copies
of the evacuated instances (`Destroying instance as it has been evacuated from this host but still exists in the
hypervisor`). A `nova-compute` that kept running while forced down does not do it: it must be restarted. The client
shows only `Compute service nova-compute of host <host> failed to set.`; the reason above is in the API response
(`openstack --debug compute service set --up <host> nova-compute`).

> 🛑 **Destructive.** At start-up `nova-compute` destroys the local guest of every instance evacuated from this host
> and, unless the storage is shared, its local disks (`_destroy_evacuated_instances`, `nova/compute/manager.py`; the
> lab saw the source copy destroyed). Precondition: every evacuated instance is `ACTIVE` on its destination, and
> nothing is needed from the source copies. Cannot be undone.

**Kolla-Ansible:** `sudo systemctl restart kolla-nova_compute-container.service`

**Any:** restart the `nova-compute` service.

> ⚠️ **State-changing.** Lets Nova consider the service alive again from its heartbeat. Precondition: the host is
> repaired, `nova-compute` runs, has been restarted since the evacuation, and has reported (`Updated At` recent), and `openstack server migration list --host
> <host> --type evacuation` shows no `done` record. Undo: none. Forcing the service down again is step 2 of
> [fence before you evacuate](../recovery/compute-host-failure-fence-before-evacuate.md), with its precondition:
> on a host that is not fenced, it allows local deletes and evacuations of instances that are alive.

```bash
openstack compute service set --up <host> nova-compute
# then, when the host is ready for new instances (not run on the lab: the service was still enabled):
openstack compute service set --enable <host> nova-compute
```

Never force down a compute whose instances still run there and that is not fenced: Nova then allows local deletes
and evacuations of instances that are alive. See
[fence before you evacuate](../recovery/compute-host-failure-fence-before-evacuate.md).

## Verification

```bash
openstack compute service list --service nova-compute --host <host> --long
# expected: enabled, up, Forced Down False, Updated At within the last few seconds
openstack resource provider trait list <node-uuid> | grep COMPUTE_STATUS_DISABLED
# expected: nothing
openstack allocation candidate list --resource VCPU=1 --resource MEMORY_MB=512 --forbidden COMPUTE_STATUS_DISABLED | grep <node-uuid>
# expected: the compute is a candidate
```

## Lab observations

OpenStack 2026.1, Kolla-Ansible 22.2.0, 3 controllers and 2 computes, Ubuntu 24.04, nested KVM, ML2/OVS, local
disks, no Cinder, 2026-10-27. All four causes produced on purpose on one compute.

- Cause 1, `systemctl stop kolla-nova_libvirt-container.service`: `nova-compute` logged `Connection event '0'
  reason 'Connection to libvirt lost: 0'` at once; the service showed `disabled`, `up`, reason `AUTO: Connection to
  libvirt lost: 0` within 3 s, with the `COMPUTE_STATUS_DISABLED` trait. A build forced to that host failed with
  `No valid host was found.` (scheduler: `Got no allocation candidates from the Placement API`). The two guests kept
  running. After `systemctl start`, `Connection event '1' reason 'None'` came 8 s later and the service was
  `enabled`, trait gone, at the next check 15 s after the start.
- Cause 2: `--disable --disable-reason` set the trait and removed the compute from the allocation candidates;
  `--enable` cleared both.
- Cause 3, `systemctl stop kolla-nova_compute-container.service`: `down` 63 s after the last heartbeat. Before
  that, `openstack server evacuate` was refused with HTTP 400 `Compute service of <host> is still in use.`. While
  `down`: resize refused with HTTP 409 `Service is unavailable at this time.`; the scheduler logged `... has not
  been heard from in a while`; a reboot was accepted (status `REBOOT`); a delete returned at once, the port was
  deleted and the guest kept running. When `nova-compute` started again, it cleared the reboot's task state at
  start-up (`Instance in transitional state (rebooting) at start-up ... clearing task state`), then received the
  queued reboot and dropped it (`Task possibly preempted`, from `reverts_task_state` on
  `UnexpectedTaskStateError`), and destroyed the deleted instance's guest 34 s after the start
  (`Destroying instance with name label '<instance-name>' which is marked as DELETED but still present on host.`).
- Cause 4, `--down` while `nova-compute` ran: `down`, `Forced Down` `True` at once, `Status` stayed `enabled`.
  An evacuation was accepted and left a `done` record; `--up` was refused (message above, through `--debug`). The
  source copy of the evacuated instance kept running on the compute until `nova-compute` was
  restarted; it was then destroyed, the record went to `completed`, and `--up` worked. Verification as written.
- Not run: `--enable` while libvirt was down, an `AUTO: ` disable over an operator reason, the trait update error,
  volume detach (no Cinder), stop of an instance, the `--enable` after `--up` (the service was still enabled), and
  forcing down a fenced host (the lab host was not fenced).

## Validity and limits

- Written from Nova 2026.1: `nova/api/openstack/compute/services.py` (enable, disable, forced down,
  `_check_for_evacuations`), `nova/servicegroup/api.py` and `drivers/db.py`, `nova/virt/libvirt/driver.py`
  (`_set_host_enabled`, `DISABLE_PREFIX`), `nova/virt/libvirt/host.py`, `nova/compute/manager.py`
  (`_update_compute_provider_status`, `_cleanup_running_deleted_instances`, `_destroy_evacuated_instances`),
  `nova/compute/api.py` (`check_instance_host`, local delete, evacuate), `nova/scheduler/filters/compute_filter.py`,
  `nova/scheduler/request_filter.py`, and `nova/conf/service.py` and `compute.py`. Reproduced on a lab (Kolla-Ansible, 2026.1, ML2/OVS, no Cinder), see *Lab observations*.
- Only `resize` (which includes cold migration) and volume detach check that the compute is up (`check_instance_host(check_is_up=True)` in `nova/compute/api.py`); delete and evacuate check it on their own. Other actions are cast over RPC without that check.
- Kolla-Ansible paths for the logs; the `openstack` commands apply to any deployment.
- Not covered: Ironic, the `[compute] consecutive_build_service_disable_threshold` weigher (see the max retries
  page), and service groups other than the database driver.

## Related

- [Compute host unreachable, instances down. Fence before you evacuate](../recovery/compute-host-failure-fence-before-evacuate.md)
- [nova-compute is down on a compute, but its instances are still running](../recovery/nova-compute-down-instances-still-running.md)
- [No valid host was found, although capacity exists](no-valid-host-with-free-capacity.md): a disabled service is
  one of its causes.
- [Instance stuck in deleting](../lifecycle/instance-stuck-in-deleting.md)

---

*Provenance: written by agent:ozarland-author, from the Nova 2026.1 documentation and source. Reviewed in its pull
request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=scheduling/compute-service-disabled-or-forced-down.md&title=Feedback%3A+scheduling/compute-service-disabled-or-forced-down.md):
your release, your deployment method, and what happened.
