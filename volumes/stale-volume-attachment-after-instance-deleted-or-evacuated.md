---
title: Stale volume attachment. The volume is still in-use for an instance that was deleted or evacuated
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [cinder, nova, libvirt]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any multinode
  network: any
  storage: any Cinder backend
  os: any
risk: destructive
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/admin/manage-volumes.html
  - https://docs.openstack.org/nova/2026.1/cli/nova-manage.html#volume-attachment-commands
  - https://docs.openstack.org/api-ref/block-storage/v3/#attachments
  - https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/volume.html
  - https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/volume-attachment.html
  - https://bugs.launchpad.net/nova/+bug/2004555
authored_by: agent:ozarland-author
last_verified: 2026-10-24
---

# Stale volume attachment. The volume is still in-use for an instance that was deleted or evacuated

## Symptom

One of these, on a volume that is not multi-attach:

- The instance was deleted, but the volume is still `in-use` and "Attached to" names the deleted instance. The user
  can neither delete the volume nor attach it to another instance.
- The instance was evacuated to another compute and runs there, but Cinder lists more than one attachment for the
  volume, or an attachment whose connector names the old host.

```text
$ openstack volume show <volume-id> -c status -c attachments
| status      | in-use                                         |
| attachments | [{'server_id': '<server-id>', ...}]            |
$ openstack server show <server-id>
No server with a name or ID of '<server-id>' exists.
```

## Impact

One volume and its owner: the volume cannot be reused or deleted, and its quota stays consumed. The data is intact.
The risk is in the fix: removing an attachment that a running instance still uses, or forcing the status of a volume
that a hypervisor still has open, can give the volume two writers and corrupt it.

## Quick checks

With admin credentials:

```bash
# 1. Cinder: the volume and every attachment record for it (3.27 or later)
openstack volume show <volume-id> -c status -c attachments
cinder --os-volume-api-version 3.27 attachment-list --all-tenants 1 --volume-id <volume-id>
openstack --os-volume-api-version 3.27 volume attachment show <attachment-id>   # Properties: the connector, with the host
#    Not "openstack volume attachment list --volume-id": with python-openstackclient 9.0.0 the filter is ignored
#    and every attachment of the cloud is listed. Check the Volume ID column before you delete anything.

# 2. Nova: does the instance exist, where does it run, and which attachment does it use?
openstack server show <server-id> -c status -c OS-EXT-SRV-ATTR:host -c OS-EXT-SRV-ATTR:instance_name
openstack --os-compute-api-version 2.89 server volume list <server-id>

# 3. Was it deleted? The last actions on the instance
openstack server event list <server-id>
```

**Kolla-Ansible:** on the host named by the stale attachment's connector, and on the instance's current host:

```bash
sudo docker exec nova_libvirt virsh list --all
sudo docker exec nova_libvirt virsh domblklist <instance-name>
```

**Any:** the same `virsh` commands on those hosts.

## Diagnosis

Classify each attachment record of the volume:

| What you find | Meaning |
|---|---|
| The instance does not exist in Nova, and no domain on its last host uses the volume | Stale: left by the deletion |
| The instance exists, and the attachment ID is the one `server volume list` shows | In use: keep it |
| The instance exists on host B, and this attachment's connector names host A, with another ID than Nova's | Stale: left by the evacuation |

`nova-manage volume_attachment show <server-id> <volume-id> --json` also prints the attachment ID that Nova's
database holds for the pair, including the connection information the API does not show.

**Kolla-Ansible:** `sudo docker exec nova_api nova-manage volume_attachment show <server-id> <volume-id> --json` on
a controller. For an instance that was deleted it answers `No volume Block Device Mapping with id <volume-id>`.

**Any:** run it where `nova-manage` has access to the Nova database.

For the deletion case, the `nova-api` and `nova-compute` logs at the time of the deletion usually show why Nova could
not tell Cinder: Cinder unreachable, or the compute down while the instance was deleted.

What Nova 2026.1 did on a lab, which narrows down when this happens:

- `cinder-volume` down during a normal deletion: the instance stayed `deleting` (7 minutes in the longest run), and
  the deletion and the detach finished once `cinder-volume` was back. No stale attachment.
- The compute down during the deletion: `nova-api` deleted the instance from its database (`instance's host <host>
  is down, deleting from database`), and Cinder released the volume once it could. No stale attachment either, but
  **the guest kept running on the host** with the volume open until `nova-compute` came back and destroyed it
  (`Destroying instance with name label '<instance-name>' which is marked as DELETED but still present on host`),
  while Cinder already showed the volume `available`. The destroyed domain stayed defined, and the host kept an
  iSCSI session to the volume. Check `virsh list --all` and the storage sessions on the instance's last host
  before you trust an `available` status after such a deletion.

## Possible causes

### Cause 1: the instance was deleted while Cinder or its compute was unreachable

Nova deleted the instance but the detach in Cinder failed or was never sent. Tell it by the instance missing from Nova
with the attachment still present, and errors towards Cinder in the Nova logs at the time of the deletion.

### Cause 2: an evacuation left the source host's attachment behind

Evacuation rebuilds the instance on another host and attaches its volumes there. If the rebuild failed part-way or
the old attachment was not removed, Cinder keeps a record whose connector names the source host. Tell it by the
instance running on another host than the one in the connector, with an attachment ID that Nova does not use.

## Resolution

Before any step: the host named in the stale attachment's connector must not run a domain that uses the volume
(`virsh domblklist`). For an evacuation, the source host must be fenced or its domain destroyed; see
[fence before you evacuate](../recovery/compute-host-failure-fence-before-evacuate.md).

### For cause 1: delete the stale attachment

> ⚠️ **State-changing.** Deleting the attachment record releases the volume in Cinder and removes the export for
> that attachment on the storage backend. The volume's data is not touched. Precondition: the instance no longer
> exists in Nova, and no domain on its last host uses the volume. Undo: none needed; the volume can be attached again
> normally.

```bash
openstack --os-volume-api-version 3.27 volume attachment delete <attachment-id>
openstack volume show <volume-id> -c status      # expected: available, once no attachment remains
```

### For cause 2: remove the old host's attachment, keep Nova's

Preferred, documented in the Nova administration guide for 2026.1: a hard reboot makes Nova check the instance's
attachments against Cinder and remove the dangling ones from both databases.

> ⚠️ **State-changing.** A hard reboot powers the instance off and on: the guest loses what was not written to disk.
> Agree on a time with the instance's owner.

```bash
openstack server reboot --hard <server-id>
```

If a reboot is not acceptable, delete only the attachment that Nova does not use:

> ⚠️ **State-changing.** Same effect as for cause 1, on one record. Precondition: its ID differs from the one in
> `server volume list` (compute API 2.89), and its connector names the old host, which is fenced. Cinder refuses to
> delete an attachment that the instance still uses, with HTTP 409 (bug 2004555); on deployments without that
> protection, the check of the ID is all that stands between you and detaching a running instance's disk.

```bash
openstack --os-volume-api-version 3.27 volume attachment delete <attachment-id>
```

### When Nova's own attachment is wrong after the evacuation

The instance runs, but its attachment carries outdated connection information (for example a connector from the old
host), and `nova-manage volume_attachment show` confirms it. Nova documents a refresh with `nova-manage`, available
since Xena:

> ⚠️ **State-changing.** The instance must be stopped and unlocked. The refresh disconnects the host's volume
> connections, deletes the Cinder attachment, creates a new one and updates Nova's database. The instance is down
> until it is started again.

```bash
openstack server stop <server-id>
```

**Kolla-Ansible:** not working as written on Kolla-Ansible 22.2.0 (see Lab observations). Each line below failed on a
lab, for a different reason:

- `docker exec nova_compute nova-manage volume_attachment get_connector --json` fails as the container's default user
  (`FailedToDropPrivileges: privsep helper command exited non-zero`); it works with `sudo docker exec -u root`.
- `docker exec nova_api nova-manage volume_attachment refresh …` runs as root and fails with `[Errno 13] Permission
  denied: '/dev/shm/<host>_nova-api_qmanager'` (the host's `fs.protected_regular`).
- Run as the `nova` user, the refresh times out waiting for the compute's reply (Kolla-Ansible sets
  `use_queue_manager = true`). **It has then already removed the volume's connection and attachment**: the volume is
  left `reserved` with an attachment that has no connection information, and the instance starts without the
  volume. Repairing that took the last resort below, a hard reboot and a new attach; see the
  [volume stuck in attaching, reserved or detaching](volume-stuck-in-attaching-or-detaching.md) runbook.
- With `use_queue_manager = false` for this command only (a second `--config-file`), the compute's reply arrived.
  The full refresh was not completed on the lab.

```bash
# On the compute that hosts the instance: its connector
sudo docker exec -u root nova_compute nova-manage volume_attachment get_connector --json > connector.json
# On a controller, with that file copied there: give it to the nova_api container and refresh
sudo docker cp connector.json nova_api:/tmp/connector.json
```

> 🛑 **Destructive on Kolla-Ansible 22.2.0.** With `use_queue_manager = true` (Kolla-Ansible's setting), the
> `refresh` below timed out on a lab after it had already removed the volume's connection and its Cinder attachment.
> The volume was left `reserved`, Nova's attachment had no connection information, and the instance started without
> its disk; `openstack server remove volume` was refused. Repair path that worked, with the volume in no domain
> (`virsh domblklist`): `openstack volume set --state available --detached <volume-id>` (the last resort below, with
> its precondition), then `openstack server reboot --hard <server-id>`, which removed Nova's dangling record, then a
> normal `openstack server add volume <server-id> <volume-id>`. For the compute's reply to arrive, the command needed
> `use_queue_manager = false` for this one command, through a second configuration file; the full refresh was still
> not completed on the lab. Precondition: the instance stopped and unlocked, and a time agreed with its owner, who
> may lose the volume until the repair.

```bash
# Not lab-verified: see the warning above. The second --config-file holds
# [oslo_messaging_rabbit] use_queue_manager = false, copied into the container like connector.json
sudo docker exec -u nova nova_api nova-manage --config-file /etc/nova/nova.conf --config-file /tmp/nm.conf \
  volume_attachment refresh <server-id> <volume-id> /tmp/connector.json
```

**Any:**

```bash
nova-manage volume_attachment get_connector --json > connector.json        # on the compute
nova-manage volume_attachment refresh <server-id> <volume-id> connector.json
```

```bash
openstack server start <server-id>
```

### Last resort: the volume stays in-use with no attachment left

> 🛑 **Destructive.** `openstack volume set --state` and `--detached` only change Cinder's database record, with no
> check of the real attachment (the client's own help says so). If any hypervisor still has the volume open, a new
> attachment gives it a second writer and corrupts the data. Precondition: no attachment record left, and on every
> host named by the volume's past attachments, `virsh domblklist` shows no domain using it (for RBD volumes,
> `rbd status <pool>/volume-<volume-id>` shows no watcher). Undo: set the status back; data written by two writers
> cannot be recovered without a backup.

```bash
openstack volume set --state available --detached <volume-id>
```

## Verification

```bash
openstack volume show <volume-id> -c status -c attachments
# deleted instance: available, no attachment
# evacuated instance: in-use, one attachment, to the instance on its current host
cinder --os-volume-api-version 3.27 attachment-list --all-tenants 1 --volume-id <volume-id>
openstack --os-compute-api-version 2.89 server volume list <server-id>
```

For an evacuated instance, `virsh domblklist <instance-name>` on its current host lists the volume, and the
instance's guest sees its disk.

## Lab observations

On a lab: OpenStack 2026.1, Kolla-Ansible 22.2.0, all-in-one (one compute), Cinder LVM over iSCSI,
python-openstackclient 9.0.0.

- **Deletion case.** Real outages did not leave a stale attachment (see Diagnosis). The symptom was produced by
  creating a Cinder attachment that Nova did not know (`volume attachment create --connect` with the host's
  connector, then `volume attachment complete`) and deleting the instance:
  - the volume stayed `in-use`, attached to the deleted server;
  - `server show` answered `No Server found`;
  - `cinder attachment-list` showed one `attached` record;
  - `volume attachment delete` made the volume `available`, with no attachment and no iSCSI session left.
- **409 protection.** A `volume attachment delete` of an attachment Nova used was refused:
  `409 … ConflictNovaUsingAttachment`.
- **Hard reboot.** With Cinder's attachment gone and Nova still listing the volume, a hard reboot removed Nova's record:
  `Removing stale volume attachment '<attachment-id>' from instance for volume '<volume-id>'`.
- **Last resort.** `volume set --state available --detached` worked as written. Cinder refused
  `volume set --state in-use` on a volume with no attachment.
- **`nova-manage volume_attachment show`.** Worked as written for an existing instance; its `attachment_id` was the one
  `server volume list` showed.
- **`get_connector` and `refresh`.** Failed as written on Kolla-Ansible; see the resolution section.

Not run: the evacuation case (it needs two computes) and RBD.

## Validity and limits

Written from the upstream documentation and client source for 2026.1: the Nova volume administration guide (hard
reboot clean-up, `nova-manage volume_attachment`), the Block Storage API reference (attachment statuses, the HTTP 409
protection), and the OpenStack client's `volume set` and `volume attachment` commands. Partly observed on an
all-in-one lab (see Lab observations), not on a multinode platform. A run of the evacuation case on a multinode
lab, and of the `refresh` procedure on a deployment where it completes, are still wanted.

Per release:

- `nova-manage volume_attachment show`, `get_connector` and `refresh` exist from Nova 24.0.0 (Xena).
- Cinder attachment records need volume API 3.27 or later; the attachment ID in `server volume list` needs compute
  API 2.89 or later (Xena and later).
- The HTTP 409 protection on attachment deletion comes from bug 2004555; check that your release and deployment
  carry it before relying on it.
- The hard-reboot clean-up of dangling attachments is documented in the 2026.1 Nova guide; check the guide of your
  release before relying on it on an older one.

Not covered: multi-attach volumes, boot volumes of deleted instances (`delete_on_termination` decides whether they
are kept), host-side clean-up of leftover iSCSI sessions or multipath devices, and direct database edits.

## Related

- [Volume stuck in attaching, reserved or detaching](volume-stuck-in-attaching-or-detaching.md)
- [Compute host unreachable, instances down. Fence before you evacuate](../recovery/compute-host-failure-fence-before-evacuate.md)
- [Stale records: stuck instances, orphan ports and attachments, phantom allocations](../lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md)
- Nova, [Manage volumes](https://docs.openstack.org/nova/2026.1/admin/manage-volumes.html) and
  [nova-manage volume_attachment](https://docs.openstack.org/nova/2026.1/cli/nova-manage.html#volume-attachment-commands)
- Block Storage API, [attachments](https://docs.openstack.org/api-ref/block-storage/v3/#attachments)

---

*Provenance: written by agent:ozarland-author, from upstream documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*
