---
title: Instance snapshot never finishes. Task state image_pending_upload or image_uploading, image queued or saving
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [nova, glance, libvirt, ceph]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: any
  storage: local instance disks, or Ceph RBD for instances and images
  os: any
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#libvirt.snapshots_directory
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#workarounds.disable_libvirt_livesnapshot
  - https://docs.openstack.org/glance/2026.1/user/statuses.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/shared-services/glance-guide.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Instance snapshot never finishes. Task state image_pending_upload or image_uploading, image queued or saving

## Symptom

After `openstack server image create --name <name> <server-id>`, the instance keeps a snapshot task state for much
longer than the size of its disk explains, and the image does not become `active`:

```text
$ openstack server show <server-id> -c status -c OS-EXT-STS:task_state
| OS-EXT-STS:task_state | image_uploading |      (or image_snapshot_pending, image_snapshot, image_pending_upload)
| status                | ACTIVE          |
$ openstack image show <image-id> -c status -c size
| status | saving |                              (or queued)
```

Other actions on the instance are refused while the task state is set. Or the snapshot fails: the task state
clears and the image disappears.

## Impact

One instance: no other action (reboot, resize, migrate, a new snapshot) until the task state clears. During a cold
snapshot, the instance is suspended. The snapshot is written to the compute's disk first, and an upload of a large
disk loads the compute's network, the Glance API and its storage.

## Quick checks

Read-only. With admin credentials:

```bash
# 1. The instance's task state, host, and the snapshot action
openstack server show <server-id> -c OS-EXT-STS:task_state -c OS-EXT-STS:power_state -c OS-EXT-SRV-ATTR:host
openstack server event list <server-id>              # createImage, its start time and result

# 2. The image: status, size so far, when it was created
openstack image show <image-id> -c status -c size -c created_at -c updated_at

# 3. Is the compute up?
openstack compute service list --service nova-compute --host <compute>
```

On the compute, the steps logged for the server ID, and the space where the snapshot is written:

**Kolla-Ansible:**

```bash
sudo grep <server-id> /var/log/kolla/nova/nova-compute.log | grep -iE 'snapshot|upload' | tail -n 20
sudo docker exec nova_compute df -h /var/lib/nova/instances
sudo docker exec nova_compute ls -la /var/lib/nova/instances/snapshots/
```

**Any:** the `nova-compute` log, and the file system of `[libvirt] snapshots_directory` (default
`$instances_path/snapshots`).

## Diagnosis

### How a snapshot runs (Nova 2026.1)

1. **API** (`nova/compute/api.py`): creates the image record in Glance (status `queued`), sets the task state to
   `image_snapshot_pending`, and sends `snapshot_instance` to the compute.
2. **Compute** (`nova/compute/manager.py`): logs `instance snapshotting`, task state `image_snapshot`.
3. **libvirt driver** (`nova/virt/libvirt/driver.py`, `snapshot`):
   - **Direct snapshot** (instance disks and images both on Ceph RBD): a Ceph snapshot and clone in the images pool;
     nothing is uploaded through Glance. If Ceph refuses, the driver logs `Performing standard snapshot because
     direct snapshot failed: <error>` and falls back to the standard path.
   - **Standard snapshot:** task state `image_pending_upload`; the driver logs `Beginning live snapshot process`
     (the default for a running instance) or `Beginning cold snapshot process` (instance shut off, LVM, encrypted
     ephemeral storage, or `[workarounds] disable_libvirt_livesnapshot`; a running instance is suspended for it).
     It writes the disk to a temporary directory under `snapshots_directory` on the compute, logs
     `Snapshot extracted, beginning image upload`, sets `image_uploading`, and uploads the file to Glance (image
     status `saving`).
4. **End:** `Snapshot image upload complete`, then `Took <n> seconds to snapshot the instance on the hypervisor.`;
   the task state clears, the image is `active`.

On any error, `nova-compute` logs `Failed to snapshot image` with a traceback, **deletes the image** in Glance
(`delete_image_on_error`), and clears the task state. A snapshot that failed leaves no image behind; a snapshot that
is stuck leaves a `queued` or `saving` image.

When `nova-compute` starts, it clears every snapshot task state left on its instances (`_init_instance`; the log
line `Instance in transitional state <state> at start-up clearing task state` is at debug level). The image in
Glance is not cleaned up then.

### Where it stops

| Last state | Last compute log line | Where to look |
|---|---|---|
| `image_snapshot_pending` | none for this server | The request did not reach the compute: `nova-compute` down or busy, message bus |
| `image_pending_upload` | `Beginning live/cold snapshot process` | Writing the snapshot on the compute: disk space under `snapshots_directory`, disk speed, a large disk |
| `image_uploading` | `Snapshot extracted, beginning image upload` | The upload to Glance: network from the compute, the Glance API, Glance's storage |
| cleared, image gone | `Failed to snapshot image` with a traceback | The traceback names the error |

## Possible causes

### Cause 1: a large disk on a slow path

Nothing is broken; the copy takes time. The image `size` (check 2) grows, or the temporary file under
`snapshots_directory` grows. With Kolla-Ansible, HAProxy's client and server timeouts for the Glance API are 6 hours
(`haproxy_glance_api_client_timeout`, `haproxy_glance_api_server_timeout`): an upload longer than that is cut.

### Cause 2: no space on the compute for the snapshot

The whole disk is written under `snapshots_directory` before the upload, on the same file system as the instances
by default. A full file system makes the extract fail (the snapshot fails, the image is deleted), and can also stop
the instances that write to their disks there ([compute disk full](compute-disk-full-instance-paused.md)).

### Cause 3: Glance cannot store the image

Glance's storage is full or unreachable: the file store (with Kolla-Ansible's default `glance_backend_file`, the
`glance` Docker volume on the first `glance-api` host), or the Ceph images pool. The upload fails or hangs; the
Glance API log has the error.

### Cause 4: `nova-compute` stopped or restarted during the snapshot

The task state was cleared at restart, or stays set if `nova-compute` is down. The image remains `queued` or `saving`
in Glance, empty or partial.

## Resolution

### For cause 1: wait, and watch progress

Repeat quick check 2 and the `ls` of the snapshots directory. If the image `size` grows, let it finish. For large
instances, plan snapshots where the network and Glance allow, or snapshot their data volumes with Cinder instead.

### For causes 2 and 3: free space, then snapshot again

Free space on the compute (cause 2) or on Glance's storage (cause 3), from the error in the traceback. The failed
snapshot cleaned up after itself: run the snapshot again.

### For cause 4: clear the task state, delete the stale image

If `nova-compute` is down, start it: it clears the task state at start-up. Then the image left behind:

> ⚠️ **State-changing.** Deletes the image record and any partial data in Glance. Precondition: no snapshot of this
> instance is running any more (the task state is cleared, the compute log shows no upload in progress), and the
> image is the stale one (`queued` or `saving`, created at the time of the failed snapshot). Undo: none; run the
> snapshot again.

```bash
openstack image delete <image-id>
```

If the task state stays set with `nova-compute` up and no snapshot in progress on the compute, restart `nova-compute`
on that compute: it clears the task state of its instances (above).

> ⚠️ **State-changing.** Restarts `nova-compute` on that compute; running instances are not affected; operations in
> progress on its instances fail. Undo: none needed.

**Kolla-Ansible:** `sudo docker restart nova_compute`

**Any:** restart the `nova-compute` service.

A cold snapshot interrupted after the instance was suspended can leave it `SUSPENDED` or `SHUTOFF`: start it again
once the task state is clear (`openstack server resume` or `openstack server start`).

## Verification

```bash
openstack server show <server-id> -c status -c OS-EXT-STS:task_state      # ACTIVE (or as before), no task state
openstack image show <image-id> -c status -c size                         # active, with a size, for a new snapshot
openstack image list --status queued ; openstack image list --status saving   # no leftovers of this instance
```

To check that a snapshot is usable, boot a test instance from it, then delete the instance.

## Validity and limits

- Written from Nova 2026.1 (`nova/compute/api.py` `snapshot`, `nova/compute/manager.py` `_snapshot_instance`,
  `delete_image_on_error`, `_init_instance`; `nova/virt/libvirt/driver.py` `snapshot`; `snapshots_directory`,
  `disable_libvirt_livesnapshot`) and Kolla-Ansible 2026.1 (Glance role defaults and backends). Not reproduced on a
  lab.
- Not covered: snapshots of instances booted from volume (they create Cinder volume snapshots, not a disk upload),
  Glance image import, Glance multi-store, and drivers other than libvirt.

## Related

- [Compute disk full. An instance that writes is frozen](compute-disk-full-instance-paused.md)
- [Stale records](stale-records-stuck-instances-orphan-ports-attachments.md)
- Nova 2026.1: [configuration reference](https://docs.openstack.org/nova/2026.1/configuration/config.html)
- Glance 2026.1: [image statuses](https://docs.openstack.org/glance/2026.1/user/statuses.html)

---

*Provenance: written by agent:ozarland-author, from the Nova and Kolla-Ansible 2026.1 documentation and source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=lifecycle/snapshot-stuck-in-image-uploading.md&title=Feedback%3A+lifecycle/snapshot-stuck-in-image-uploading.md):
your release, your deployment method, and what happened.
