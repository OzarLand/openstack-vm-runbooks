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
last_verified: 2026-10-21
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
openstack --os-volume-api-version 3.27 volume attachment list --all-projects --volume-id <volume-id>
openstack --os-volume-api-version 3.27 volume attachment show <attachment-id>   # Properties: the connector, with the host

# 2. Nova: does the instance exist, where does it run, and which attachment does it use?
openstack server show <server-id> -c status -c OS-EXT-SRV-ATTR:host -c OS-EXT-SRV-ATTR:instance_name
openstack --os-compute-api-version 2.89 server volume list <server-id>

# 3. Was it deleted? The last actions on the instance
openstack server event list <server-id>
```

**Kolla-Ansible:** on the host named by the stale attachment's connector, and on the instance's current host:

```bash
docker exec nova_libvirt virsh list --all
docker exec nova_libvirt virsh domblklist <instance-name>
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

**Kolla-Ansible:** `docker exec nova_api nova-manage volume_attachment show <server-id> <volume-id> --json` on a
controller.

**Any:** run it where `nova-manage` has access to the Nova database.

For the deletion case, the `nova-api` and `nova-compute` logs at the time of the deletion usually show why Nova could
not tell Cinder: Cinder unreachable, or the compute down while the instance was deleted.

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

**Kolla-Ansible:**

```bash
# On the compute that hosts the instance: its connector
docker exec nova_compute nova-manage volume_attachment get_connector --json > connector.json
# On a controller, with that file copied there: give it to the nova_api container and refresh
docker cp connector.json nova_api:/tmp/connector.json
docker exec nova_api nova-manage volume_attachment refresh <server-id> <volume-id> /tmp/connector.json
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
openstack --os-volume-api-version 3.27 volume attachment list --all-projects --volume-id <volume-id>
openstack --os-compute-api-version 2.89 server volume list <server-id>
```

For an evacuated instance, `virsh domblklist <instance-name>` on its current host lists the volume, and the
instance's guest sees its disk.

## Validity and limits

Written from the upstream documentation and client source for 2026.1, not observed on a platform yet: the Nova
volume administration guide (hard reboot clean-up, `nova-manage volume_attachment`), the Block Storage API reference
(attachment statuses, the HTTP 409 protection), and the OpenStack client's `volume set` and `volume attachment`
commands. A lab run or an operator's confirmation is wanted, in particular of the evacuation case.

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
