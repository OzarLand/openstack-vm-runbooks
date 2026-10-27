---
title: Instance snapshot never finishes. Task state image_pending_upload or image_uploading, image queued or saving
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, glance, libvirt, ceph]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: any
  storage: local instance disks, or Ceph RBD for instances and images
  os: any
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-27, note: "standard snapshot path, local disks and Glance file store, upload slowed with tc: cause 1 (slow upload) and cause 4 (nova-compute restarted during the upload: task state cleared, image left queued, deleted) run, and an upload failure; not run: causes 2 and 3, the cold snapshot, the Ceph direct snapshot and rbd checks, the restart for a task state stuck with nova-compute up" }
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#libvirt.snapshots_directory
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#workarounds.disable_libvirt_livesnapshot
  - https://docs.openstack.org/glance/2026.1/user/statuses.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/shared-services/glance-guide.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
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
snapshot, the instance is suspended. On the standard path, the snapshot is written to the compute's disk first, and
an upload of a large disk loads the compute's network, the Glance API and its storage.

## Quick checks

Read-only. With admin credentials:

```bash
# 1. The instance's task state, host, and the snapshot action
openstack server show <server-id> -c OS-EXT-STS:task_state -c OS-EXT-STS:power_state -c OS-EXT-SRV-ATTR:host
openstack server event list <server-id>              # createImage and its start time (its result stays Success
                                                     # even when the upload fails later)

# 2. The image: status, size (shown only once active), when it was created
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
2. **Compute** (`nova/compute/manager.py`): sets the task state `image_snapshot`, then waits for a slot: at most
   `[DEFAULT] max_concurrent_snapshots` (default 5) snapshots run at once on a compute. In its slot, it logs
   `instance snapshotting`.
3. **libvirt driver** (`nova/virt/libvirt/driver.py`, `snapshot`), in this order:
   - logs `Beginning live snapshot process` (the default for a running instance) or `Beginning cold snapshot
     process` (instance shut off, LVM, encrypted ephemeral storage, or `[workarounds] disable_libvirt_livesnapshot`;
     a running instance is suspended with a managed save for it), and sets `image_pending_upload`, then
     `image_uploading`;
   - **direct snapshot** (instance disks and images both on Ceph RBD): a Ceph snapshot and clone in the images pool;
     nothing is uploaded through Glance, and the image stays `queued` until Nova records its location. If Ceph
     refuses, the driver logs `Performing standard snapshot because direct snapshot failed: <error>`, sets
     `image_pending_upload` again, and takes the standard path;
   - **standard snapshot:** writes the disk to a temporary directory under `snapshots_directory` on the compute, logs
     `Snapshot extracted, beginning image upload`, sets `image_uploading`, and uploads the file to Glance (image
     status `saving`).
4. **End:** `Snapshot image upload complete`, then `Took <n> seconds to snapshot the instance on the hypervisor.`;
   the task state clears, the image is `active`.

On any error, `nova-compute` **deletes the image** in Glance (`delete_image_on_error`) and clears the task state,
logging `Successfully reverted task state from <task state> on failure for instance.`; the error itself is in the
lines before it (for an upload, `Error contacting glance server '<url>' for 'upload', retrying.` then `done
trying.`, with tracebacks). On the standard path (local disks, or a direct snapshot that fell back) an error is not
logged as `Failed to snapshot image`: read the lines before `Successfully reverted task state …`. On the direct
(Ceph RBD) path, a failure is logged as `Failed to snapshot image` (ERROR, with a traceback); the lab did not run that
path. A snapshot that failed leaves no image behind; a snapshot that is stuck leaves a `queued` or `saving` image.

When `nova-compute` starts, it clears every snapshot task state left on its instances (`_init_instance`; the log
line `Instance in transitional state <state> at start-up clearing task state` is at debug level). The image in
Glance is not cleaned up then.

### Where it stops

| Last state | Last compute log line | Where to look |
|---|---|---|
| `image_snapshot_pending` | none for this server | The request did not reach the compute: `nova-compute` down or busy, message bus |
| `image_snapshot` | none for this server yet | Waiting for a slot: `max_concurrent_snapshots` other snapshots run on this compute |
| `image_pending_upload` | `Performing standard snapshot because direct snapshot failed`, or `Beginning live/cold snapshot process` without Ceph | Writing the snapshot on the compute: disk space under `snapshots_directory`, disk speed, a large disk |
| `image_uploading`, image `queued` | `Beginning live/cold snapshot process`, no `Snapshot extracted` line (Ceph RBD) | The direct snapshot is stuck in Ceph: Ceph health, the images pool |
| `image_uploading`, image `saving` | `Snapshot extracted, beginning image upload` | The upload to Glance: network from the compute, the Glance API, Glance's storage |
| cleared, image gone | `Successfully reverted task state from ... on failure for instance.`, errors before it | The errors before it name the cause (standard path) |
| cleared, image gone | `Failed to snapshot image` (ERROR, traceback) | The direct (Ceph RBD) snapshot failed; the traceback names the cause |

## Possible causes

### Cause 1: a large disk on a slow path

Nothing is broken; the copy takes time. While the snapshot is `image_pending_upload`, the temporary file under
`snapshots_directory` grows. Once `Snapshot extracted, beginning image upload` is logged, the file is complete and the
upload has no progress indicator in the API: the image stays `saving` with no `size` until it turns `active`, and
`glance-api-uwsgi.log` logs the `PUT /v2/images/<image-id>/file` line only when the request ends. With Kolla-Ansible, HAProxy's client and server timeouts for the Glance API are 6 hours
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
in Glance, empty or partial. The temporary directory under `snapshots_directory`, with the full extracted disk, is
left behind on the compute.

## Resolution

### For cause 1: wait, and watch progress

Repeat quick checks 1 and 2 and the `ls` of the snapshots directory. If the temporary file grows, or the upload
started (`Snapshot extracted`) and the task state is still `image_uploading` with no error in the compute log, let it
finish. For large
instances, plan snapshots where the network and Glance allow, or snapshot their data volumes with Cinder instead.

### For causes 2 and 3: free space, then snapshot again

Free space on the compute (cause 2) or on Glance's storage (cause 3), from the error in the traceback. The failed
snapshot cleaned up after itself: run the snapshot again.

### For cause 4: clear the task state, delete the stale image

If `nova-compute` is down, start it: it clears the task state at start-up. Then find the image left behind, and check
that it is the one of this snapshot:

```bash
openstack image show <image-id> -c status -c size -c owner -c created_at -c properties
# owner: the instance's project; created_at: the time of the failed snapshot;
# properties: instance_uuid matching <server-id>; size not growing between two runs
```

> 🛑 **Destructive.** Deletes the image record and its data in Glance. It belongs to the instance's project, not only
> to this instance: another `queued` or `saving` image may be another upload still running, by this project or another
> one. Precondition: no snapshot of this instance is running any more (task state cleared, no upload in progress in
> the compute log), and the checks above all match this server. Cannot be undone; run the snapshot again afterwards.

```bash
openstack image delete <image-id>
```

Then remove the temporary directory the interrupted snapshot left on the compute:

```bash
sudo docker exec nova_compute ls -la /var/lib/nova/instances/snapshots/      # tmp<random>/ with one file each
```

> 🛑 **Destructive.** Deletes the extracted copy of the instance's disk. Precondition: no snapshot is running on this
> compute (no instance with a snapshot task state on it, no `Snapshot extracted` without `upload complete` since the
> last start of `nova-compute`), and only `tmp*` directories older than the last start of `nova_compute`
> (`sudo docker inspect -f '{{.State.StartedAt}}' nova_compute`); a running snapshot reads its file from there.
> Cannot be undone.

```bash
sudo docker exec -u root nova_compute rm -rf /var/lib/nova/instances/snapshots/<tmp-dir>
```

With a direct snapshot (Ceph), deleting the `queued` image may not remove what Ceph already created: a snapshot or a
clone can remain in the images pool or on the instance's disk. Look, read-only, with
`rbd snap ls <pool>/<server-id>_disk` and `rbd ls -l <images-pool>`, and escalate to whoever runs Ceph rather than
deleting there.

If the task state stays set with `nova-compute` up and no snapshot in progress on the compute, restart `nova-compute`
on that compute: it clears the task state of its instances (above). First check the other instances on that compute
(`openstack server list --all-projects --host <compute> --long`): a restart interrupts their migrations, resizes and
snapshots too.

> ⚠️ **State-changing.** Restarts `nova-compute` on that compute; running instances are not affected; operations in
> progress on its instances fail. Undo: none needed.

**Kolla-Ansible:** `sudo systemctl restart kolla-nova_compute-container.service` (about 60 s on the lab: the unit's
`ExecStop` waits up to Kolla-Ansible's `docker_graceful_timeout`, 60 s by default)

**Any:** restart the `nova-compute` service.

A cold snapshot suspends a running instance with a libvirt managed save, while Nova's `vm_state` stays `active`.
Interrupted, it most likely leaves the instance `SHUTOFF`: start it with `openstack server start <server-id>` once the
task state is clear (`openstack server resume` is refused unless Nova shows it `SUSPENDED`).

## Verification

```bash
openstack server show <server-id> -c status -c OS-EXT-STS:task_state      # ACTIVE (or as before), no task state
openstack image show <image-id> -c status -c size                         # active, with a size, for a new snapshot
openstack image list --status queued ; openstack image list --status saving   # no leftovers of this instance
```

To check that a snapshot is usable, boot a test instance from it, then delete the instance.

## Lab observations

Kolla-Ansible 22.2.0 (2026.1) all-in-one, local instance disks (qcow2), Glance file store, a CirrOS instance, the
upload slowed with `tc` on the Glance API port, on 2026-10-27:

- Cause 1 (upload at 4 Mbit/s): `image_pending_upload` and `queued` for about 10 s while the temporary file grew,
  then `image_uploading` and `saving` with no `size`; `Snapshot extracted` to `Snapshot image upload complete` 108 s,
  `Took 117.55 seconds to snapshot the instance on the hypervisor.`, image `active` at 53542912 bytes. The Glance
  file store file stayed at 0 bytes during the upload.
- An upload broken by the network (packets dropped): `Error contacting glance server '<url>' for 'upload',
  retrying.` three times, `done trying.`, `Successfully reverted task state from image_uploading on failure for
  instance.`; Glance `Failed to upload image data due to internal error: OSError: unable to receive chunked part`;
  the image was deleted (`DELETE` 204). No `Failed to snapshot image` line (standard path). `server event list` showed the
  `createImage` event, its `compute_snapshot_instance` result `Success`.
- Cause 4: `systemctl restart kolla-nova_compute-container.service` 20 s into the upload took 61 s; Glance logged
  `unable to receive chunked part` 16 s after the stop; `nova-compute` logged `Starting compute node` 5 s after the
  restart returned and the task state was clear within 11 s after that; the image was left `queued` (no `size`), with `instance_uuid`, `owner` and `created_at` matching. `image delete`
  worked; a new snapshot was `active` and an instance booted from it. A 52 MB `tmp<random>/` directory was left under
  `/var/lib/nova/instances/snapshots/` and removed by hand.

## Validity and limits

- Lab-run for causes 1 and 4 and for an upload failure on the standard snapshot path (see *Lab observations*). Not
  run: cause 2 (compute disk full), cause 3 (Glance store full or unreachable), the cold snapshot, the direct
  snapshot on Ceph RBD and its `rbd` checks, the `max_concurrent_snapshots` wait, the restart of `nova-compute` with
  the task state stuck while it is up.
- Written from Nova 2026.1 (`nova/compute/api.py` `snapshot`, `nova/compute/manager.py` `_snapshot_instance`,
  `delete_image_on_error`, `_init_instance`; `nova/virt/libvirt/driver.py` `snapshot`; `snapshots_directory`,
  `disable_libvirt_livesnapshot`) and Kolla-Ansible 2026.1 (Glance role defaults and backends).
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
