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
confirmed_by: []
upstream_refs:
  - https://opendev.org/openstack/glance/src/branch/stable/2026.1/glance/api/v2/image_data.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/image/glance.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/virt/libvirt/driver.py
  - https://opendev.org/openstack/kolla-ansible/src/branch/stable/2026.1/ansible/roles/glance/templates/glance-api.conf.j2
authored_by: agent:ozarland-author
last_verified: 2026-10-26
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
| `Error contacting glance server '<server>' for '<method>', retrying.` (then `done trying.`) with a traceback | ERROR | nova-compute, nova-api | Nova could not reach Glance (connection error, timeout). After the last attempt the build fails with `GlanceConnectionFailed`. |
| `Download image error` with a traceback | ERROR | nova-compute | A download through a special handler (the RBD handler when `[glance] enable_rbd_download` is on) failed; Nova falls back to downloading through the Glance API. |
| `Successfully transferred using <scheme>` | INFO | nova-compute | A special-handler download (for example `rbd`) succeeded. |
| `Error during RBD image export: <error>` | ERROR | nova-compute | Exporting an image from Ceph RBD to a local file failed (`CouldNotFetchImage`). |
| `Image signature verification failed for image <image-id>` | ERROR | nova-compute | With signature verification on, the downloaded data did not match. |
| `Beginning live snapshot process` / `Beginning cold snapshot process` | INFO | nova-compute | A snapshot started (live: the instance keeps running; cold: it is paused or stopped meanwhile). |
| `Snapshot extracted, beginning image upload` | INFO | nova-compute | The disk was copied locally; the upload to Glance starts. The time between this line and the next is the upload. |
| `Snapshot image upload complete` | INFO | nova-compute | Done. |

Sources: `nova/image/glance.py`, `nova/virt/libvirt/driver.py`. All with `[instance: <uuid>]` where an instance is
involved.

A snapshot that never logs `Snapshot image upload complete` stopped during the upload: look for an upload failure in
Glance's log at that time (table above), and for errors in `nova-compute.log` after `Snapshot extracted`.

```bash
# On the compute
grep -n "<server-id>" /var/log/kolla/nova/nova-compute.log | grep -E "snapshot|Snapshot|glance"
```

## Validity and limits

- Every message is quoted from Glance and Nova 2026.1. **None was captured on a lab.** A lab request asks for real
  samples.
- The status codes are the ones `image_data.py` raises for each case.
- Store driver messages (`glance_store` for file, RBD, Swift, S3) and the interoperable import workflow (staging,
  tasks) are not covered.

## Related

- [Volume stuck in creating or error after create](../volumes/volume-stuck-in-creating-or-error-after-create.md) (volumes from images)
- [Compute disk full](../lifecycle/compute-disk-full-instance-paused.md)
- [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md)
- [Logs index](README.md)

---

*Provenance: written by agent:ozarland-author, from Glance and Nova 2026.1 source code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Freading-glance-logs.md&title=Feedback%3A+logs%2Freading-glance-logs.md):
your release, your deployment method, and what happened.
