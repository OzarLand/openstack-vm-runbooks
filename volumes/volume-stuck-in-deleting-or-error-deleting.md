---
title: Volume stuck in deleting, in error_deleting, or back to available after a delete
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [cinder, ceph, rabbitmq]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: any
  storage: any Cinder backend; Ceph RBD details where named
  os: any
risk: destructive
confirmed_by: []
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
Invalid volume: Volume <volume-id> status must be available or error or error_restoring or error_extending or
error_managing and must not be migrating, attached, belong to a group, have snapshots, awaiting a transfer, or be
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
- A volume with copy-on-write clones (volumes or images created from its snapshots) is renamed `<volume-name>.deleted`
  and kept until its clones are gone; or, when it has snapshots, the driver flattens the dependent images first
  (`Image <pool>/<image>@<snap> is dependent on the image <volume>.`), which takes time for large clones.
- With `enable_deferred_deletion`, the image is moved to the RBD trash and deleted after `deferred_deletion_delay`.

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

### Cause 5: refused by the API

Attached, snapshots, migrating, in a group or transfer: the `Invalid volume` message names the conditions.

## Resolution

### For cause 1: bring `cinder-volume` back, then delete again

Start the service ([RabbitMQ](../recovery/rabbitmq-partition-or-alarm-rpc-timeouts.md) if the bus is the cause). A
volume left in `deleting` is not retried by Cinder on its own: set it to `error`, then delete again.

> ⚠️ **State-changing.** `volume set --state error` changes Cinder's record only, not the backend. Undo:
> `openstack volume set --state <previous-state> <volume-id>`.

```bash
openstack volume set --state error <volume-id>
```

> 🛑 **Destructive.** Deletes the volume and its data on the backend. Precondition: the owner confirms the data is not
> needed or is backed up; the volume is attached nowhere (cause 4 ruled out). Cannot be undone.

```bash
openstack volume delete <volume-id>
```

### For cause 2: fix the backend, then delete again

Fix what the traceback names, then delete again (the 🛑 above applies). A volume in `error_deleting` is not in the
list of states the API accepts: set it to `error` first, as for cause 1.

### For cause 3: wait

Follow the log. On Ceph, `rbd status <pool>/volume-<volume-id>` shows its watchers and `rbd children` its clones
(read-only).

### For cause 4: find who still uses the volume, never force it

On Ceph, `rbd status <pool>/volume-<volume-id>` lists the watchers: the address of each client. Find the compute and
the instance that still has the volume open (`virsh domblklist` on that compute), and detach it there through Nova
([stale volume attachment](stale-volume-attachment-after-instance-deleted-or-evacuated.md)). Delete only once no
watcher remains.

Do not delete a volume that a guest still writes to by forcing states or removing the backend image by hand: the guest
keeps writing to storage that Cinder considers free.

### For cause 5: remove what the API names

Detach it, delete its snapshots (or use `--cascade`), wait for the migration, or take it out of its group, then delete
again (the 🛑 above applies). `--force` drops only the status condition.

## Verification

```bash
openstack volume show <volume-id>                 # No volume with a name or ID of '<volume-id>' exists.
openstack quota show --usage <project>            # volumes and gigabytes in use went down
```

On Ceph: `rbd ls <pool> | grep <volume-id>` shows nothing, or only a `.deleted` image kept for its clones.

## Validity and limits

- Written from Cinder 2026.1 (`cinder/volume/api.py` `delete`, `cinder/volume/manager.py` `delete_volume`,
  `cinder/volume/drivers/rbd.py` `delete_volume`) and python-openstackclient 2026.1. Not reproduced on a lab.
- Not covered: volumes in groups or consistency groups, transfers, backends other than Ceph in detail, and the
  database purge of deleted rows.

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
