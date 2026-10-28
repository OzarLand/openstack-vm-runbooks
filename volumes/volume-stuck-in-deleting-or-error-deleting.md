---
title: Volume stuck in deleting, in error_deleting, or back to available after a delete
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [cinder, ceph, rabbitmq]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: any
  storage: any Cinder backend; Ceph RBD details where named
  os: any
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-27, note: "Cinder LVM: cause 1 (cinder-volume stopped with systemctl; queued delete ran at start, no reset needed, so the reset step was not run for it), cause 2 (LV held open: error_deleting, reset to error and delete), cause 5 (snapshot, attachment, --force, --cascade); not run: all Ceph RBD steps (causes 3 and 4 on Ceph, rbd status/children/ls/trash), encrypted volumes (cause 6), a lost RabbitMQ message, --force on a deleting volume" }
upstream_refs:
  - https://docs.openstack.org/cinder/2026.1/admin/manage-volumes.html
  - https://docs.openstack.org/cinder/2026.1/admin/troubleshoot.html
  - https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/volume.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Volume stuck in deleting, in error_deleting, or back to available after a delete

## Symptom

After `openstack volume delete <volume-id>`, the volume does not go away. One of:

```text
$ openstack volume show <volume-id> -c status -c attachments -c os-vol-host-attr:host
| status | deleting |          (for longer than the backend needs)
| status | error_deleting |
| status | available |         (the delete was accepted, then the volume came back)
```

Or the delete is refused at once:

```text
Invalid volume: Volume status must be available or error or error_restoring or error_extending or error_managing
and must not be migrating, attached, belong to a group, have snapshots, awaiting a transfer, or be
disassociated from snapshots after volume transfer.
```

## Impact

One volume: it keeps its space on the backend and counts against the project's quota until it is deleted. No instance
is affected, unless the volume is still in use somewhere (cause 4), which is exactly when it must not be deleted.

## Quick checks

Read-only. With admin credentials:

```bash
# 1. Status, attachments, snapshots, the cinder-volume service that owns it
openstack volume show <volume-id> -c status -c attachments -c os-vol-host-attr:host -c migration_status
openstack volume snapshot list --all-projects --volume <volume-id>

# 2. Is that cinder-volume service up?
openstack volume service list --service cinder-volume
```

The `cinder-volume` log of the service named in `os-vol-host-attr:host`, for the volume ID:

**Kolla-Ansible:** `sudo grep <volume-id> /var/log/kolla/cinder/cinder-volume.log | tail -n 30`

**Any:** the same `grep` on the `cinder-volume` log.

## Diagnosis

### How a delete runs (Cinder 2026.1)

1. **API** (`cinder/volume/api.py`, `delete`): accepts the delete only if the volume is in `available`, `error`,
   `error_restoring`, `error_extending` or `error_managing`, is not attached, not migrating, not in a group, and has
   no snapshots (unless `--cascade`). It sets `deleting` and sends the request to the volume's `cinder-volume` service.
   With `--force` (admin), the status condition is dropped; the attachment and snapshot conditions are not.
2. **`cinder-volume`** (`cinder/volume/manager.py`, `delete_volume`) asks the backend driver to delete the volume, then
   removes the record and logs `Deleted volume successfully.`
3. On failure:
   - the driver says the volume is busy (`VolumeIsBusy`): `cinder-volume` logs `Unable to delete busy volume.` and puts
     the volume back to **`available`**;
   - any other error: the volume goes to **`error_deleting`**, with the traceback in the log.

A volume that stays in `deleting` means step 2 never ran or never finished: the request did not reach a running
`cinder-volume`, or the backend call hangs.

### Ceph RBD (`cinder/volume/drivers/rbd.py`)

- A volume that still has an RBD **watcher** (a client that has it open, such as a QEMU process on a compute) raises
  `ImageBusy`, logged as `ImageBusy error raised while deleting RBD volume`, and the volume goes back to `available`.
- A volume that other volumes were cloned from (it has a `<volume-name>.clone_snap` snapshot) is renamed
  `<volume-name>.deleted` and kept until those clones are gone; when it has other snapshots with dependent images, the
  driver flattens them first (`Image <pool>/<image>@<snap> is dependent on the image <volume>.`), which takes time for
  large clones.
- The driver first tries to remove the image; the RBD trash is a fallback when the removal fails (snapshots
  remain). `enable_deferred_deletion` sets that fallback's delay (`deferred_deletion_delay`) and starts Cinder's
  periodic trash purge (every `deferred_deletion_purge_interval`, 60 s by default). Without it (the default), a
  trashed image stays in the RBD trash, and keeps its space, until someone or a Ceph trash purge schedule purges it.
- **Encrypted volumes:** the API deletes the volume's encryption key from the key manager **before** sending the
  delete. From then on the volume cannot be decrypted, even if it later comes back `available`. If the key manager
  fails, the volume goes to `error_deleting` with `Unable to delete encryption key for volume` in the API's answer and
  log, and nothing reaches `cinder-volume`.

### When `cinder-volume` restarts

A delete is recorded as a cleanable operation before it is sent. A message still queued is consumed when
`cinder-volume` comes back, and a delete that `cinder-volume` had taken and not finished is run again at start-up
(`CleanableManager` in `cinder/manager.py`, logging `Initiating service <id> cleanup`; `_do_cleanup` in
`cinder/volume/manager.py` deletes volumes found in `deleting`).

## Possible causes

### Cause 1: `cinder-volume` was down or the request was lost

The service that owns the volume is `down`, was restarting, or the message bus lost the request. The log has nothing
for the volume after the delete.

### Cause 2: the backend failed the delete

The backend refused or failed (unreachable array, permission, a Ceph error): `error_deleting` and a traceback.

### Cause 3: the backend is slow

Flattening clones on Ceph, or a large delete on an array: the log shows the operation in progress.

### Cause 4: the volume is still in use

The backend sees the volume open (an RBD watcher, an exported LUN with a session): `Unable to delete busy volume.`, back
to `available`. Often a compute that still has the volume attached although Cinder no longer shows the attachment
([stale volume attachment](stale-volume-attachment-after-instance-deleted-or-evacuated.md)).

On LVM, a logical volume still open on the `cinder-volume` host does not give `available` back: `lvremove` fails with
`Logical volume cinder-volumes/volume-<volume-id> in use.` and the volume goes to `error_deleting` (as cause 2).

### Cause 5: refused by the API

Attached, snapshots, migrating, in a group or transfer: the `Invalid volume` message names the conditions.

### Cause 6: the key manager refused to delete the encryption key

Encrypted volume, `error_deleting`, `Unable to delete encryption key for volume` from the API, nothing in
`cinder-volume`'s log.

## Resolution

### For cause 1: bring `cinder-volume` back, then delete again

Start the service ([RabbitMQ](../recovery/rabbitmq-partition-or-alarm-rpc-timeouts.md) if the bus is the cause), then
**wait**: the delete may still be queued, and an interrupted one is run again at start-up. Read the service's log for
the volume: `Initiating service … cleanup`, then `Deleted volume successfully.` Only if the volume is still in
`deleting` once the service is up and its log shows no delete in progress for it, set it to `error` and delete again.

> ⚠️ **State-changing.** `volume set --state error` changes Cinder's record only, not the backend. Precondition: no
> delete of this volume in progress in the `cinder-volume` log (a reset during a delete that is still running races
> it). For an encrypted volume, the API deleted the key before the delete was sent (causes 1 to 5): the data is not
> recoverable whatever the status says. In cause 6 the key deletion failed, so the key is still there. Undo:
> `openstack volume set --state <previous-state> <volume-id>`.

```bash
openstack volume set --state error <volume-id>
```

> 🛑 **Destructive.** Deletes the volume and its data on the backend. Precondition: the owner confirms the data is not
> needed or is backed up; the volume is attached nowhere (cause 4 ruled out). Cannot be undone.

```bash
openstack volume delete <volume-id>
```

`openstack volume delete --force <volume-id>` (admin) *(not run on the lab on a `deleting` volume)* has the same effect
as the reset and the delete together, with the same 🛑 and both preconditions: the ⚠️ one (no delete of this volume in progress) and the 🛑 one.

### For cause 2: fix the backend, then delete again

Fix what the traceback names, then delete again (the 🛑 above applies). A volume in `error_deleting` is not in the
list of states the API accepts: set it to `error` first, as for cause 1.

### For cause 3: wait

Follow the log, and do not reset the state or delete again while the flatten runs: it would race the delete Cinder is
running. On Ceph *(not run on the lab)*, `rbd status <pool>/volume-<volume-id>` shows its watchers and `rbd children` its clones
(read-only).

### For cause 4: find who still uses the volume, never force it

On Ceph *(not run on the lab)*, `rbd status <pool>/volume-<volume-id>` lists the watchers: the address of each client. Find the compute and
the instance that still has the volume open (`virsh domblklist` on that compute), and detach it there through Nova
([stale volume attachment](stale-volume-attachment-after-instance-deleted-or-evacuated.md)). Delete only once no
watcher remains.

Do not delete a volume that a guest still writes to by forcing states or removing the backend image by hand: the guest
keeps writing to storage that Cinder considers free.

### For cause 5: remove what the API names

Detach it, delete its snapshots, wait for the migration, or take it out of its group, then delete again (the 🛑 above
applies). `--cascade` deletes the volume **and its snapshots** in one call: the 🛑 then covers the snapshots too.
`--force` drops only the status condition.

### For cause 6: fix the key manager, then delete again

*(Not run on the lab.)*

Fix the key manager (Barbican, or the configured backend) from the API log, then reset to `error` and delete again
(the ⚠️ and 🛑 above apply).

## Verification

```bash
openstack volume show <volume-id>                 # No Volume found for <volume-id>
openstack quota show --usage <project>            # volumes and gigabytes in use went down
```

On Ceph *(not run on the lab)*: `rbd ls <pool> | grep <volume-id>` shows nothing, or only a `.deleted` image kept for its clones; `rbd trash
ls <pool>` shows whether it waits in the trash: without `enable_deferred_deletion`, Cinder never purges it.

## Lab observations

Kolla-Ansible 22.2.0 (2026.1) all-in-one, Cinder LVM (thin, iSCSI), python-openstackclient 9.0.0, on 2026-10-27,
with 1 GB volumes:

- Cause 5: a volume with a snapshot, and an attached one, were refused at once with the page's `Invalid volume: ...`
  message (400). With `--force` on the attached one: `Invalid volume: Volume  must not be migrating, attached, belong
  to a group, have snapshots, awaiting a transfer, or be disassociated from snapshots after volume transfer.` (the
  status condition dropped, as the page says). `volume delete --cascade` removed the volume and its snapshot in 6 s.
- Cause 1: `systemctl stop kolla-cinder_volume-container.service`, then `volume delete`: `deleting`, nothing in the
  `cinder-volume` log; the service showed `down` in `volume service list` within 53 s. After `systemctl
  start`: `Initiating service 2 cleanup`, `Service 2 cleanup completed.`, then the queued delete ran (its request ID
  is the one of the API call) and `Deleted volume successfully.` 15 s after the start; no reset needed.
- Cause 2 (LVM volume held open on the host): `error_deleting` 37 s after the delete, the `cinder-volume` log with
  `ProcessExecutionError` from `lvremove` and `Stderr: '  Logical volume cinder-volumes/volume-<volume-id> in use.'`.
  A new `volume delete` was refused (`Invalid volume: Volume status must be available or error ...`). After closing
  the device, `volume set --state error` and `volume delete`: gone in 6 s.
- `openstack volume show` of a deleted volume: `No Volume found for <volume-id>`. `quota show --usage` showed the
  volumes and gigabytes in use back to 0.
- Not run: the Ceph RBD parts (causes 3 and 4 on Ceph, `rbd status`, `rbd children`, trash), encrypted volumes (cause
  6), `volume delete --force` of a `deleting` volume, a lost RabbitMQ message.

## Validity and limits

- Written from Cinder 2026.1 (`cinder/volume/api.py` `delete`, `cinder/volume/manager.py` `delete_volume`,
  `cinder/volume/drivers/rbd.py` `delete_volume`) and python-openstackclient 2026.1. Reproduced on an all-in-one lab
  with Cinder LVM for causes 1, 2 and 5 (*Lab observations*); the Ceph RBD and encrypted-volume parts were not.
- Not covered: volumes in groups or consistency groups, transfers, backends other than Ceph in detail (for LVM,
  leftovers show in `sudo lvs <vg> | grep volume-<volume-id>`), and the database purge of deleted rows.

## Related

- [Stale volume attachment](stale-volume-attachment-after-instance-deleted-or-evacuated.md)
- [Volume stuck in attaching or detaching](volume-stuck-in-attaching-or-detaching.md)
- [Stale records](../lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md)
- Cinder 2026.1: [manage volumes](https://docs.openstack.org/cinder/2026.1/admin/manage-volumes.html)

---

*Provenance: written by agent:ozarland-author, from the Cinder and python-openstackclient 2026.1 source and
documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=volumes/volume-stuck-in-deleting-or-error-deleting.md&title=Feedback%3A+volumes/volume-stuck-in-deleting-or-error-deleting.md):
your release, your deployment method, and what happened.
