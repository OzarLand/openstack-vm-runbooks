---
title: Reading the Glance logs, and the image lines in Nova's logs (boot, snapshot)
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [glance, nova]
deployment_methods: [kolla-ansible, any]
environment:
  topology: multinode
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-27, note: "Glance file store: the page's greps run, a snapshot and Glance stopped during a boot captured; not injected: the Glance upload failures (store full, quota, size cap, permissions, signature) and the RBD and signature lines of Nova" }
upstream_refs:
  - https://opendev.org/openstack/glance/src/branch/stable/2026.1/glance/api/v2/image_data.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/image/glance.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/virt/libvirt/driver.py
  - https://opendev.org/openstack/kolla-ansible/src/branch/stable/2026.1/ansible/roles/glance/templates/glance-api.conf.j2
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Reading the Glance logs, and the image lines in Nova's logs (boot, snapshot)

Glance is on the path of every boot from image (Nova downloads the image to the compute, unless it is cloned inside a
shared Ceph cluster) and of every snapshot (Nova uploads it). Glance's own log says whether the store accepted or
served the data; Nova's log on the compute says how the transfer went from its side. Messages are quoted from Glance
and Nova 2026.1.

## Where the logs are

**Kolla-Ansible:** `/var/log/kolla/glance/glance-api.log` (`log_file` in `glance-api.conf.j2`), with the HTTP
requests in `glance-api-uwsgi.log`, on the nodes that run `glance_api`. Nova's side is in `nova-compute.log` on the
compute.

**Any:** `/var/log/glance/` or the journal.

Nova's calls to Glance carry Nova's request id as Glance's global request id, so the Glance lines of one boot or
snapshot are found from the instance action's request id
([where the logs are](where-openstack-logs-are-and-how-to-read-a-line.md#request-id-and-global-request-id)).

```bash
grep "<nova-request-id>" /var/log/kolla/glance/glance-api.log
grep "<image-id>" /var/log/kolla/glance/glance-api-uwsgi.log | tail     # PUT .../file (upload), GET .../file (download)
```

The `glance-api-uwsgi.log` lines carry no request id and their time is the start of the request; find them by image
id. Captured on a Kolla-Ansible 2026.1 lab (file store), a snapshot upload and a download at boot:

```text
[pid: 28|app: 0|req: 28/110] 192.0.2.10 () {42 vars in 1033 bytes} [Tue Oct 27 07:17:43 2026] PUT /v2/images/<image-id>/file => generated 0 bytes in 2133 msecs (HTTP/1.1 204) 4 headers in 171 bytes (1 switches on core 0)
[pid: 18|app: 0|req: 3/10] 192.0.2.10 () {40 vars in 1000 bytes} [Tue Oct 27 07:20:22 2026] GET /v2/images/<image-id>/file => generated 21692416 bytes in 69 msecs (HTTP/1.1 200) 5 headers in 217 bytes (349 switches on core 0)
```

At the default INFO level, a successful upload leaves little in `glance-api.log`; the snapshot's upload on the lab
left, with Nova's request id:

```text
2026-10-27 07:17:45.260 28 INFO glance.location [None req-<nova-request-id> <user-id> <project-id> - - default default] Image format matched and virtual size computed: 1073741824
```

## In Glance's log: upload failures

When image data is uploaded (an image create, or a Nova snapshot), Glance logs the store's refusal (ERROR unless
noted) and answers the client with an error (`glance/api/v2/image_data.py`):

| Line (message text) | Client gets | What it means |
|---|---|---|
| `Image storage media is full: <error>` | 413 | The store backend (a file system, a Ceph pool) has no room. |
| `Image exceeds the storage quota: <error>` | 413 | A Glance quota (per user or project) is reached. |
| `The incoming image is too large: <error>` | 413 | The image exceeds `image_size_cap`. |
| `Insufficient permissions on image storage media: <error>` | 503 | Glance cannot write to the store (file permissions, Ceph user capabilities). |
| `Error in store configuration. Adding images to store is disabled.` | 410 | The configured store does not accept new images. |
| `Signature verification failed for image <image-id>: <error>` | 400 | The image's signature properties do not match its data. |
| `Not allowed to upload image data for image <image-id>` (DEBUG only) | 403 | Policy refused the upload. |
| `Unable to restore image <image-id>: <error>` | — | After a failed upload, Glance could not put the image back to `queued`; it may stay `saving`. |

```bash
grep -nE "storage media is full|exceeds the storage quota|too large|Insufficient permissions|store configuration" \
  /var/log/kolla/glance/glance-api.log | tail
```

## In Nova's log: download at boot, upload at snapshot

| Line (message text) | Level | Log | What it means |
|---|---|---|---|
| `Error contacting glance server '<server>' for '<method>', retrying.` (then `done trying.`) with a traceback | ERROR | nova-compute, nova-api | Nova could not reach Glance (connection error, timeout). After the last attempt the build fails with `GlanceConnectionFailed`, and is rescheduled; the instance's fault is then `Exceeded maximum number of retries...` when no other compute is left. |
| `Download image error` with a traceback | ERROR | nova-compute | A download through a special handler (the RBD handler when `[glance] enable_rbd_download` is on) failed; Nova falls back to downloading through the Glance API. |
| `Successfully transferred using <scheme>` | INFO | nova-compute | A special-handler download (for example `rbd`) succeeded. |
| `Error during RBD image export: <error>` | ERROR | nova-compute | Exporting an image from Ceph RBD to a local file failed (`CouldNotFetchImage`). |
| `Image signature verification failed` (followed by `for image <image-id>` or `for image: <image-id>`, depending on the path) | ERROR | nova-compute | With signature verification on, the downloaded data did not match. |
| `Beginning live snapshot process` / `Beginning cold snapshot process` | INFO | nova-compute | A snapshot started (live: the instance keeps running; cold: a running instance is suspended, a managed save, and resumed afterwards). |
| `Snapshot extracted, beginning image upload` | INFO | nova-compute | The disk was copied locally; the upload to Glance starts. The time between this line and the next is the upload. With Ceph RBD ephemeral disks, Nova takes a direct snapshot in Ceph instead and never logs this line: its absence is normal there. |
| `Performing standard snapshot because direct snapshot failed: <error>` | WARNING | nova-compute | The direct (RBD) snapshot was refused (`ImageUnacceptable`, `Forbidden`); Nova falls back to extracting and uploading the disk. |
| `Failed to snapshot image` with a traceback | ERROR | nova-compute | The snapshot failed; the traceback names the cause. (`Failed to snapshot image because it was deleted`, WARNING: the image was deleted in Glance during the snapshot.) |
| `Snapshot image upload complete` | INFO | nova-compute | Done. |

Sources: `nova/image/glance.py`, `nova/virt/libvirt/driver.py`. All with `[instance: <uuid>]` where an instance is
involved.

Captured on the lab, `glance_api` stopped during a boot from an image not cached on the compute (`[glance]
num_retries = 3`: four attempts about 1 s apart, method `data`), then `nova-conductor`'s verdict on a single compute:

```text
2026-10-27 07:19:31.942 7 ERROR nova.image.glance [None req-<nova-request-id> <user-id> <project-id> - - default default] Error contacting glance server 'http://192.0.2.10:9292' for 'data', retrying.: glanceclient.exc.CommunicationError: Error finding address for http://192.0.2.10:9292/v2/images/<image-id>: Unable to establish connection to ...
2026-10-27 07:19:34.990 7 ERROR nova.image.glance [None req-<nova-request-id> <user-id> <project-id> - - default default] Error contacting glance server 'http://192.0.2.10:9292' for 'data', done trying.: glanceclient.exc.CommunicationError: ...
2026-10-27 07:19:34.996 7 ERROR nova.compute.manager [None req-<nova-request-id> <user-id> <project-id> - - default default] [instance: <server-id>] Instance failed to spawn: nova.exception.GlanceConnectionFailed: Connection to glance host http://192.0.2.10:9292 failed: ...
2026-10-27 07:19:38.739 7 ERROR nova.compute.manager [None req-<nova-request-id> <user-id> <project-id> - - default default] [instance: <server-id>] Failed to build and run instance: nova.exception.GlanceConnectionFailed: ...
2026-10-27 07:19:41.070 112 WARNING nova.scheduler.utils [None req-<nova-request-id> ...] Failed to compute_task_build_instances: Exceeded maximum number of retries. Exhausted all hosts available for retrying build failures for instance <server-id>.: nova.exception.MaxRetriesExceeded: ...
```

With Glance down before the request, `nova-api` logged the same line for `'get'` twice (`retrying.`, then
`done trying.` 1.2 s later) and answered 500 `Unexpected API Error ... <class 'nova.exception.GlanceConnectionFailed'>`.
The counts differ because Kolla-Ansible's nova role sets `[glance] num_retries` in `nova-api` to the number of
`glance-api` hosts (one on this all-in-one lab: 2 attempts), and its nova-cell role sets 3 for `nova-compute` (4
attempts). `openstack server create` fails earlier, in its own image lookup (`The image service for <region> exists
but does not have any supported versions.`).

A snapshot of a running instance, captured on a 2026.1 lab (local qcow2 disks, `nova-compute.log`):

```text
07:34:12.299 INFO nova.compute.manager [...] [instance: <server-id>] instance snapshotting
07:34:12.325 INFO nova.virt.libvirt.driver [...] [instance: <server-id>] Beginning live snapshot process
07:34:24.981 INFO nova.virt.libvirt.driver [...] [instance: <server-id>] Snapshot extracted, beginning image upload
07:34:27.804 INFO nova.virt.libvirt.driver [...] [instance: <server-id>] Snapshot image upload complete
07:34:27.805 INFO nova.compute.manager [...] [instance: <server-id>] Took 15.51 seconds to snapshot the instance on the hypervisor.
```

A shut-off instance logs `trying to snapshot a non-running instance: (state: 4 expected: 1)` (WARNING) then
`Beginning cold snapshot process`, and the same three lines after it.

A snapshot that never logs `Snapshot image upload complete` stopped during the upload: look for an upload failure in
Glance's log at that time (table above), and for errors in `nova-compute.log` after `Snapshot extracted`.

```bash
# On the compute
grep -n "<server-id>" /var/log/kolla/nova/nova-compute.log | grep -E "snapshot|Snapshot|glance"
```

## Validity and limits

- Every message is quoted from Glance and Nova 2026.1. Captured on a Kolla-Ansible 22.2.0 (2026.1) all-in-one lab
  with the Glance file store and local qcow2 disks (2026-10-27): the `glance-api-uwsgi.log` PUT and GET lines, the
  `glance.location` line, the live and cold snapshot lines (cold on a shut-off instance), and the `Error contacting
  glance server` lines (compute and API) with the build failure. Not captured: the managed save of a running
  instance, RBD direct snapshots, the WARNING/ERROR snapshot lines, the Glance upload-failure table (no store full,
  quota, size cap, permission or signature failure was injected), `Download image error`, `Successfully transferred
  using`, `Error during RBD image export` and the signature line (no Ceph, no signed image).
- The status codes are the ones `image_data.py` raises for each case.
- Store driver messages (`glance_store` for file, RBD, Swift, S3) and the interoperable import workflow (staging,
  tasks) are not covered.

## Related

- [Instance snapshot stuck in image_uploading](../lifecycle/snapshot-stuck-in-image-uploading.md)
- [Volume stuck in creating or error after create](../volumes/volume-stuck-in-creating-or-error-after-create.md) (volumes from images)
- [Compute disk full](../lifecycle/compute-disk-full-instance-paused.md)
- [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md)
- [Logs index](README.md)

---

*Provenance: written by agent:ozarland-author, from Glance and Nova 2026.1 source code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Freading-glance-logs.md&title=Feedback%3A+logs%2Freading-glance-logs.md):
your release, your deployment method, and what happened.
