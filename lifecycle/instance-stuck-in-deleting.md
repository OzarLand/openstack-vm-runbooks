---
title: Instance stuck in deleting, or in ERROR after a delete. Deleting it again does nothing
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, libvirt, cinder, neutron, placement]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: any
  storage: any
  os: any
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-26, note: "causes 1, 4 and 5 on a disposable lab: delete stuck with nova-compute paused or stopped, resumed by the restart; cause 4 reproduced with the service down from a lost heartbeat (unit stopped), local delete, guest reaped 54 s after the return; the forced-down command (--down) and its undo (--up --enable) not run; lock 409 for a member, admin bypass; causes 2 and 3 not run" }
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/reference/vm-states.html
  - https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/server.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Instance stuck in deleting, or in ERROR after a delete. Deleting it again does nothing

## Symptom

After `openstack server delete <server-id>`, the instance does not go away:

```text
$ openstack server show <server-id> -c status -c OS-EXT-STS:task_state -c OS-EXT-SRV-ATTR:host
| OS-EXT-SRV-ATTR:host  | <compute>  |
| OS-EXT-STS:task_state | deleting   |
| status                | ACTIVE     |      (or SHUTOFF, ERROR...)
```

or it ends in `ERROR` with a fault from the delete. Running `openstack server delete` again returns without error
and changes nothing: while its compute is up, Nova's API logs `Instance is already in deleting state, ignoring this
request` (`nova/compute/api.py`).

If `[DEFAULT] reclaim_instance_interval` is set on the platform (default 0, off), a delete is a soft delete: the
instance goes to `SOFT_DELETED`, not `deleting`, and is reclaimed later. That case is not covered here.

## Impact

One instance. It keeps its quota, its Placement allocation on the compute, its ports (and IP addresses) and its
volume attachments until the delete completes. If the guest was not destroyed, it may still be running on the
compute.

## Quick checks

Read-only. With admin credentials:

```bash
# 1. State, host, and the delete action
openstack server show <server-id> -c status -c OS-EXT-STS:task_state -c OS-EXT-STS:power_state \
  -c OS-EXT-SRV-ATTR:host -c OS-EXT-SRV-ATTR:instance_name -c locked -c fault
openstack server event list <server-id>

# 2. Is the instance's compute up, as Nova sees it?
openstack compute service list --service nova-compute --host <compute>
```

On the compute, does the guest still exist, and what did `nova-compute` log for it?

**Kolla-Ansible:**

```bash
sudo docker exec nova_libvirt virsh list --all --uuid | grep <server-id>
sudo grep <server-id> /var/log/kolla/nova/nova-compute.log | tail -n 30
```

**Any:** `sudo virsh list --all --uuid | grep <server-id>`, and the `nova-compute` log.

## Diagnosis

### How a delete runs (Nova 2026.1)

The API (`nova/compute/api.py`, `_delete`) chooses between two paths:

- **The compute is up:** it sets `task_state` `deleting` and sends `terminate_instance` to that compute over RPC.
  The compute (`nova/compute/manager.py`) logs `Terminating instance`, destroys the guest through libvirt
  (`Instance destroyed successfully.`), deallocates the network, detaches the volumes, deletes the Placement
  allocation, then removes the instance from the database.
- **The compute is down** (or the instance has no host, or is shelved offloaded): a **local delete**. The API logs
  `instance's host <compute> is down, deleting from database`, deallocates the ports, deletes the volume
  attachments, **deletes the volumes created with `delete_on_termination`**, removes the allocation itself, and marks the instance deleted. Nothing is done on the compute: if it comes
  back, the guest may still be defined there.

`task_state` `deleting` with the compute up means the API sent the request and the compute did not finish it.

### What the compute logs

| Message (`nova-compute` log) | Meaning |
|---|---|
| no `Terminating instance` for the server ID | The compute never received the request: message bus problem, or `nova-compute` blocked |
| `Terminating instance`, then nothing | The compute is stuck in the delete, usually in libvirt destroying a guest whose I/O hangs (storage lost) |
| `Cannot destroy instance, operation time out` | libvirt could not kill the guest in time; the instance goes to `ERROR` |
| `Error from libvirt during destroy. Code=<n> Error=<message>` | Other libvirt failure; the instance goes to `ERROR` |
| `Setting instance vm_state to ERROR` with a traceback | Any error of the delete on the compute: the traceback names the step |
| `Ignoring unknown cinder exception for volume <id>` or `Ignoring unknown exception for volume <id>` | Cinder failed to detach a volume; Nova continues the delete, and the volume may stay `in-use` |
| `Service started deleting the instance during the previous run, but did not finish. Restarting the deletion now.` | `nova-compute` restarted and resumed the delete |

## Possible causes

### Cause 1: the request never reached the compute, or `nova-compute` is stuck

A message bus incident, or `nova-compute` blocked on another operation. The compute log has no
`Terminating instance` for the server ID.

### Cause 2: the guest cannot be destroyed

The QEMU process does not die, usually because it waits on storage that no longer answers (NFS, iSCSI, Ceph lost).
libvirt reports a timeout, or `nova-compute` waits.

### Cause 3: the delete failed part way (ERROR)

A libvirt error, or a failure after the guest was destroyed (network deallocation, Placement). The instance is in
`ERROR`; deleting it again retries the whole delete.

### Cause 4: the compute is gone, but Nova still sees it up

The host died less than `service_down_time` (60 s by default) ago, or something still reports for it. Nova sends the
request to a compute that will never answer.

### Cause 5: the instance is protected

Not stuck, refused: for a non-admin, a locked instance (`locked` `True`) returns `Instance <id> is locked` (HTTP 409);
an admin's delete goes through a lock (`check_instance_lock` in `nova/compute/api.py` skips admins). An instance with
`disable_terminate` set is not deleted, for anyone, and the API only logs `instance termination disabled`.

## Resolution

### For cause 1: restart `nova-compute`, which resumes the delete

Fix the message bus first if it is the cause
([RabbitMQ](../recovery/rabbitmq-partition-or-alarm-rpc-timeouts.md),
[shared dependencies](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md)).
At start-up, `nova-compute` resumes every delete left in `deleting` on its host (`_init_instance`).

> ⚠️ **State-changing.** Restarts `nova-compute` on that compute; running instances are not affected, operations in
> progress on that compute fail. Undo: none needed.

**Kolla-Ansible:** `sudo systemctl restart kolla-nova_compute-container.service`

Not `docker restart nova_compute`: it bypasses the container's systemd unit, which then fails and re-attaches after
`RestartSec` (11 s with the Kolla defaults, 59 s on the lab, which sets `docker_client_timeout: 600`). Stopping
`nova_compute` can take up to 60 s (the unit's `ExecStop` is `docker stop nova_compute -t 60`). On a lab, `docker
restart` of a **paused** `nova_compute` only unpaused it: the old process then ran the queued delete before being
killed.

**Any:** restart the `nova-compute` service.

### For cause 2: free the guest, then delete again

*(Not run on the lab.)*

Repair the storage path if you can: the QEMU process then dies and the delete completes. A process waiting on
storage that is gone for good often cannot be killed; then the host needs a reboot, after moving its other
instances ([maintenance](../migration/moving-instances-off-a-host-for-maintenance.md)). Once the guest is gone from
`virsh list --all`, delete again (cause 3).

> ⚠️ **State-changing.** Rebooting the host stops every instance on it; move them first. When `nova-compute` starts
> again, it resumes the delete left in `deleting` (cause 1). Undo: none needed.

Do not shortcut with `virsh undefine`, by removing the instance's disks by hand, or by editing the database: Nova,
Neutron, Cinder and Placement would keep records that no longer match the host.

### For cause 3: delete again

*(Cause 3 not reproduced on the lab.)*

An instance in `ERROR` with no task state accepts a new delete, which runs every step again:

> 🛑 **Destructive.** Deletes the instance: its guest, its local disks, its ports and their addresses, its
> allocations, and the volumes created with `delete_on_termination` (other volumes are detached and kept). What is
> lost: the data on the local disks and on those volumes. Precondition: the owner confirms the instance's data is not
> needed or is backed up (a snapshot of its volumes, a copy of its data). Cannot be undone.

```bash
openstack server delete <server-id>
```

If the instance stays in `deleting` with its compute up and the guest already gone, restart `nova-compute` first
(cause 1): a new delete goes back to the same compute, and resetting the state while that compute may still be
working on the first delete runs two deletes at once. Check the compute log: a `Terminating instance` line for the
server ID with no `Instance destroyed successfully.` after it means the compute is still on it. If it still stays in `deleting` after the restart, clear the
task state, then delete again (the 🛑 above applies):

> ⚠️ **State-changing.** `server set --state error` sets the instance to `ERROR` with no task state, in the database
> only. Undo: `openstack server set --state active <server-id>`. The `delete` that follows is irreversible: the 🛑 of
> the previous step applies to it.

```bash
# not run on the lab; with python-openstackclient 10.5.0 add --auto-approve (see below)
openstack server set --state error <server-id>
openstack server delete <server-id>
```

python-openstackclient 10.5.0 asks `Do you wish to continue? [y/n]` before `server set --state` and `--auto-approve`
skips the question; without a terminal it exits without changing the state (seen on two labs, once with exit code
1). Whether earlier clients (9.0.0 on other labs) prompt or accept `--auto-approve` is not recorded here: check
`openstack server set --help`.

### For cause 4: make sure the host is down, then let Nova delete locally

If the host is really gone, wait until Nova reports its `nova-compute` `down` (quick check 2), then delete again:
the API does a local delete. If the host may come back, fence it first
([fence before you evacuate](../recovery/compute-host-failure-fence-before-evacuate.md)): a host that returns still
has the guest defined, and its `nova-compute` finds an instance it no longer knows.

> 🛑 **Destructive.** Forcing the service down affects **every instance on that compute**: any delete of them becomes
> a local delete, and they can be evacuated, by an operator or by auto-evacuation tooling. The local delete then
> removes the instance from Nova, Neutron and Cinder without touching the host: its ports and addresses are freed, its
> volume attachments deleted, its `delete_on_termination` volumes deleted and its allocation removed. If the host is
> not really fenced, the guest keeps running while its addresses and volumes are released and reused: duplicate IP
> addresses, two writers on one volume. What is lost: the instance record, the `delete_on_termination` volumes, the
> local disks; a guest is left on the host until it is reaped (below). Precondition: the host is powered off or cut
> off from the network and the storage, checked out of band (BMC, switch port), not only through Nova; the owner
> agrees; a snapshot of each `delete_on_termination` volume whose data matters. Undo: `openstack
> --os-compute-api-version 2.11 compute service set --up --enable <compute> nova-compute` restores the service, not the
> deleted instance.

```bash
# not run on the lab: cause 4 was reproduced with the service down from a lost heartbeat
openstack --os-compute-api-version 2.11 compute service set --disable --disable-reason "<why>" --down \
  <compute> nova-compute
openstack server delete <server-id>
```

When the host comes back, its `nova-compute` finds the guest of a deleted instance. With the default
`[DEFAULT] running_deleted_instance_action = reap`, every `running_deleted_instance_poll_interval` (1800 s by default,
and once shortly after `nova-compute` starts: 54 s on a lab) it logs
`Destroying instance with name label '<name>' which is marked as DELETED but still present on host.` and destroys it. Check afterwards that no guest and no leftover disk of that instance remain on the host
(`virsh list --all`, the instances directory).

### For cause 5: unlock or allow termination

The owner, or an admin, unlocks the instance (`openstack server unlock <server-id>`) and deletes it again (the 🛑 of
cause 3 applies). There is no API or CLI command to clear `disable_terminate`; changing it in the database would be a
destructive step of its own, needing a database backup and its own review, and is not covered here.

## Verification

```bash
openstack server show <server-id>                       # No server found
openstack port list --device-id <server-id>             # empty
openstack volume show <volume-id> -c status -c attachments   # for each volume it had: no attachment to it
```

On the compute, the guest is gone (`virsh list --all --uuid`). If the delete was local, check when the compute
comes back that the guest is gone there too. For ports, volumes or allocations left behind, see
[stale records](stale-records-stuck-instances-orphan-ports-attachments.md) and
[stale volume attachment](../volumes/stale-volume-attachment-after-instance-deleted-or-evacuated.md).

## Lab observations

On a lab: OpenStack 2026.1 (Nova 33.0.3), Kolla-Ansible 22.2.0, all-in-one, python-openstackclient 9.0.0.

- **Cause 1, compute paused.** `docker pause nova_compute`, then delete: `task_state` `deleting`, status `ACTIVE`, the
  guest running, no `Terminating instance`; a second delete logged `Instance is already in deleting state, ignoring
  this request` in `nova-api.log`.
- **Cause 1, compute stopped before the request.** `systemctl stop` of the unit (60 s), delete while the service was
  still `up`, then `systemctl start`: `Service started deleting the instance during the previous run, but did not
  finish. Restarting the deletion now.` 9 s after the start, and the instance was gone 24 s after it.
- **Cause 4.** Unit stopped, service `down` 51 s later, delete: `instance's host <compute> is down, deleting from
  database` (WARNING), ports and allocation removed at once, **the guest still running**. After `systemctl start`,
  the first `running_deleted_instance` run, 54 s later, logged `Destroying instance with name label '<name>' which is
  marked as DELETED but still present on host.`, destroyed and undefined the domain and deleted its files.
- **Cause 5.** A member user's delete of its locked instance: `409 … Instance <id> is locked`. The admin's delete of
  the same locked instance went through, as the page says.
- On Kolla-Ansible, `docker stop nova_compute` is undone by the systemd unit after `RestartSec` (59 s on
  the lab, 11 s with the Kolla defaults), before Nova's 60 s `service_down_time`: stop the unit to reproduce cause 4.

Not run: causes 2 and 3, `disable_terminate`, `server set --state error`, and the forced-down command of cause 4
(`compute service set ... --down`) with its undo (`--up --enable`): cause 4 was reproduced with the service down
from a lost heartbeat (unit stopped).

## Validity and limits

- Written from Nova 2026.1 (`nova/compute/api.py` `_delete`, `_local_delete`; `nova/compute/manager.py`
  `terminate_instance`, `_shutdown_instance`, `_init_instance`; `nova/virt/libvirt/driver.py` `_destroy`) and
  python-openstackclient 2026.1. Causes 1, 4 and 5 reproduced on an all-in-one lab (see Lab observations); causes 2
  and 3 not. Cause 4 was reproduced with the service down from a lost heartbeat (unit stopped); the forced-down
  command (`--down`) and its undo (`--up --enable`) were not run.
- Not covered: soft delete (`reclaim_instance_interval`, instances in `SOFT_DELETED`), Ironic instances, and down
  cells.

## Related

- [Stale records](stale-records-stuck-instances-orphan-ports-attachments.md): the periodic clean-up of instances,
  ports, attachments and allocations.
- [Compute host unreachable. Fence before you evacuate](../recovery/compute-host-failure-fence-before-evacuate.md)
- [Stale volume attachment after an instance was deleted](../volumes/stale-volume-attachment-after-instance-deleted-or-evacuated.md)
- Nova 2026.1: [VM states and transitions](https://docs.openstack.org/nova/2026.1/reference/vm-states.html)

---

*Provenance: written by agent:ozarland-author, from the Nova and python-openstackclient 2026.1 documentation and
source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=lifecycle/instance-stuck-in-deleting.md&title=Feedback%3A+lifecycle/instance-stuck-in-deleting.md):
your release, your deployment method, and what happened.
