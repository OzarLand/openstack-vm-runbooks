---
title: Volume stuck in attaching, reserved or detaching. The attach or detach never finishes
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [cinder, nova, libvirt]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any multinode
  network: any
  storage: any Cinder backend
  os: any
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, all-in-one, Cinder LVM over iSCSI", date: 2026-10-24, note: "causes 1, 2 and 4 reproduced, resolutions run; the 409 protection, the hard reboot and volume set observed; cause 3 not reproduced" }
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/admin/manage-volumes.html
  - https://docs.openstack.org/api-ref/block-storage/v3/#attachments
  - https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/volume.html
  - https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/volume-attachment.html
  - https://bugs.launchpad.net/nova/+bug/2004555
authored_by: agent:ozarland-author
last_verified: 2026-10-24
---

# Volume stuck in attaching, reserved or detaching. The attach or detach never finishes

## Symptom

A user attached or detached a volume. Minutes later the volume is still in a transitional status, and retrying the
request fails with an error about the volume's status (for a volume that is not multi-attach, Cinder only starts
an attach from `available` and a detach from `in-use`):

```text
$ openstack volume show <volume-id> -c status -c attachments
| status      | attaching |                      # or reserved, or detaching
...
```

The server's action list shows an `attach_volume` or `detach_volume` event that failed or never finished:

```text
$ openstack server event list <server-id>
| <request-id> | <server-id> | detach_volume | <date> |
```

## Impact

One volume and the instance it was being attached to or detached from. The data on the volume is not at risk while
nothing forces its state. The risk comes from the fixes: forcing a Cinder status without checking the hypervisor can
leave a volume attached to two instances at once, and two writers corrupt a filesystem.

## Quick checks

With admin credentials, from any host with the OpenStack client:

```bash
# 1. Status of the volume and of its attachment records (Cinder side). Attachment records need 3.27 or later.
openstack volume show <volume-id> -c status -c attachments
cinder --os-volume-api-version 3.27 attachment-list --all-tenants 1 --volume-id <volume-id>
#    Not "openstack volume attachment list --volume-id": with python-openstackclient 9.0.0 the filter is ignored
#    and every attachment of the cloud is listed. Check the Volume ID column of whatever you list.

# 2. What Nova thinks is attached (the attachment ID column needs compute API 2.89 or later)
openstack --os-compute-api-version 2.89 server volume list <server-id>

# 3. The last attach or detach action on the instance, and its error
openstack server event list <server-id>
openstack server event show <server-id> <request-id>

# 4. Are the services involved up?
openstack volume service list
openstack compute service list --service nova-compute
```

Then, on the compute that hosts the instance (`openstack server show <server-id> -c OS-EXT-SRV-ATTR:host -c OS-EXT-SRV-ATTR:instance_name`):

**Kolla-Ansible:**

```bash
# 5. Does the hypervisor see the volume as a disk of the instance?
sudo docker exec nova_libvirt virsh domblklist <instance-name>
```

**Any:**

```bash
virsh domblklist <instance-name>
```

Compare the four views: Cinder's volume status, Cinder's attachment records, Nova's list, and the hypervisor. The
fix depends on which of them disagree.

## Diagnosis

### Cinder

- Volume `reserved` or `attaching` with one attachment in status `reserved` or `attaching`: Nova started the attach
  and did not complete it. Attachment statuses are listed in the Block Storage API reference.

  While Nova is still trying, the attach is not stuck yet: the last event of `openstack server event list` has no
  result, and `server volume list` already lists the volume. On a lab with `cinder-volume` stopped, Nova retried the
  attachment update every 60 s. When the service came back within that time the attach completed by itself
  (`in-use`, event `Success`). After about six minutes Nova gave up (`CinderConnectionFailed`) and released the
  volume by itself (`available`, event `Error`). Wait for the event's result before changing anything.
- Volume `detaching`: Nova started the detach and Cinder never received the end of it. If `nova-compute` was down,
  the request waits on the message bus and runs when the service is back (observed on a lab: `available` within
  20 s of the restart). The compute service still shows `up` for about a minute after it stopped.
- `openstack volume service list` shows the `cinder-volume` service of the volume's backend `down`: Cinder cannot
  have finished anything.

Read the `cinder-volume` log around the time of the request, searching for the volume ID:

**Kolla-Ansible:** `/var/log/kolla/cinder/cinder-volume.log` on the hosts that run `cinder_volume`.

**Any:** the `cinder-volume` service log on the hosts that run it.

### Nova

Read the `nova-compute` log of the instance's host for the request ID from `openstack server event list`:

**Kolla-Ansible:** `/var/log/kolla/nova/nova-compute.log` on the compute.

**Any:** the `nova-compute` service log on the compute.

What to look for:

- an exception from `os_brick` (iSCSI login, multipath, RBD map) during attach: the host could not connect to the
  storage;
- a detach that timed out waiting for the guest: libvirt asked the guest to release the device and the guest did not.
  Nova 2026.1 tries 8 times, about 25 s each, logs `DeviceDetachFailed: Device detach failed for vdb: Run out of
  retry while detaching device`, and puts the Cinder status back to `in-use` itself (event `Error`). It has already
  removed the device from the domain's persistent configuration (`Successfully detached device vdb … from the
  persistent domain config`), while the running domain still has it;
- no trace of the request at all: the request never reached the compute (service down, message bus problem).

### Hypervisor

`virsh domblklist` tells you the truth about the instance: the volume is either one of its disks or not. Trust it
over both databases.

## Possible causes

| Cause | How to tell |
|---|---|
| 1. `cinder-volume` was down or unreachable during the request | Volume service `down` now or at the time; errors or nothing in the `cinder-volume` log |
| 2. `nova-compute` failed or restarted during the request | Compute service `down`, or a restart of the agent in its log at that time |
| 3. The compute could not connect to the storage (attach) | `os_brick` exception in `nova-compute.log`; the volume is not in `domblklist` |
| 4. The guest did not release the device (detach) | Detach timeout in `nova-compute.log`; the volume is still in `domblklist` |

## Resolution

Fix the service first (causes 1 and 2), then make the records match the hypervisor. Never force a status before you
know what the hypervisor has.

### For causes 1 and 2: bring the service back

> ⚠️ **State-changing.** Restarting `cinder-volume` or `nova-compute` interrupts the operations in progress on that
> service (other attaches, detaches, volume creations or migrations), which may then need to be retried. Running
> instances are not affected.

**Kolla-Ansible:**

```bash
sudo systemctl restart kolla-cinder_volume-container.service     # on the host running cinder_volume
sudo systemctl restart kolla-nova_compute-container.service      # on the compute
```

**Any:** restart the `cinder-volume` or `nova-compute` service with your service manager.

Then continue with the subsection that matches what `domblklist` shows.

### For cause 3, or an attach that never reached the hypervisor: release the reservation

The volume is **not** in `domblklist`, Nova does not list it (`server volume list`), and the last attach event has a
result (see Diagnosis: an attach in progress also looks like this, with Nova listing the volume).

> ⚠️ **State-changing.** Deleting the attachment record releases the volume in Cinder and removes the export on the
> storage backend for that attachment. The volume's data is not touched. Precondition: the volume is not one of
> the instance's disks (`virsh domblklist`) and Nova does not list it. If Nova still uses the attachment, Cinder
> refuses the deletion with HTTP 409 (bug 2004555); use `openstack server remove volume` in that case.

```bash
openstack --os-volume-api-version 3.27 volume attachment delete <attachment-id>
openstack volume show <volume-id> -c status      # expected: available, once no attachment remains
```

Then fix what made the connection fail (`os_brick` error in the log) before the user retries the attach.

### For cause 4: the guest did not release the device

The volume **is** still in `domblklist`, and Cinder shows `detaching`, or `in-use` again once Nova has given up
(see Diagnosis).

1. Ask the instance's owner to stop using the disk inside the guest (unmount the filesystem, stop the application),
   or to stop the instance if that is acceptable.
2. If Cinder still shows `detaching`, put the Cinder status back to what the hypervisor shows, so that the detach
   can be requested again. Skip this step if Nova already set it back to `in-use`.

   > 🛑 **Destructive.** `openstack volume set --state` only changes Cinder's database record, with no check of the
   > real attachment (the client's own help says so). Set `in-use` only while `domblklist` shows the volume attached
   > to this instance; setting it on a volume that is not attached hides the problem, and setting `available` on a
   > volume that is attached lets a second instance attach it and write to it at the same time, which corrupts the
   > data. Precondition: `domblklist` checked a minute ago, and no other attach or detach running on this volume.
   > Undo: set the status that matches the hypervisor again.

   ```bash
   openstack volume set --state in-use <volume-id>
   ```

   Cinder refuses this on a volume with no attachment record (`Cannot reset-state to in-use because volume does not
   have any attachments`): a protection against the case the warning describes.

3. Request the detach again, through Nova, which detaches on the hypervisor and then in Cinder:

   > ⚠️ **State-changing.** Detaches the volume from the instance. Anything in the guest still using the disk loses
   > it.

   ```bash
   openstack server remove volume <server-id> <volume-id>
   ```

If the guest still does not release the device, stopping the instance and detaching it while stopped avoids the
guest altogether.

### When the hypervisor detached it but the records did not follow

The volume is **not** in `domblklist`, but Nova or Cinder still show it attached. The Nova administration guide for
2026.1 documents that a hard reboot synchronises the instance's volume attachments with Cinder and removes the
dangling ones from both databases.

What the hard reboot does depends on the records (observed on a lab):

- Cinder no longer has the attachment that Nova lists: Nova removes it (`Removing stale volume attachment '<id>'
  from instance for volume '<volume-id>'` in `nova-compute.log`), and the volume is no longer listed.
- Both Nova and Cinder still have it, with its connection information: the hard reboot **attaches the volume
  again**, since the domain is rebuilt from Nova's records. To finish a detach, then run
  `openstack server remove volume <server-id> <volume-id>`.
- Nova's record has no connection information (for example after a failed `nova-manage volume_attachment refresh`):
  the hard reboot leaves the volume out, `server remove volume` is refused while the volume is `reserved` and
  fails with `VolumeDriverNotFound: Could not find a handler for None volume` once it is `in-use`. What worked on a
  lab, with the volume in no domain: the last resort of the
  [stale volume attachment](stale-volume-attachment-after-instance-deleted-or-evacuated.md) runbook
  (`volume set --state available --detached`), then this hard reboot, which removed Nova's record, then a normal
  attach.

> ⚠️ **State-changing.** A hard reboot powers the instance off and on: the guest loses what was not written to disk.
> Agree on a time with the instance's owner.

```bash
openstack server reboot --hard <server-id>
```

If the volume record is still `detaching` with no attachment left after that, see
[stale volume attachment](stale-volume-attachment-after-instance-deleted-or-evacuated.md), which covers forcing a
volume back to `available`.

## Verification

```bash
openstack volume show <volume-id> -c status -c attachments
# expected: in-use with one attachment to the right server, or available with no attachment
cinder --os-volume-api-version 3.27 attachment-list --all-tenants 1 --volume-id <volume-id>
openstack --os-compute-api-version 2.89 server volume list <server-id>
# the attachment ID Nova lists is the one Cinder lists
```

On the compute, `virsh domblklist <instance-name>` agrees with both.

## Lab observations

On a lab: OpenStack 2026.1, Kolla-Ansible 22.2.0, all-in-one, Cinder LVM over iSCSI, CirrOS instances,
python-openstackclient 9.0.0.

| Produced | Views during | Resolution | After |
|---|---|---|---|
| Cause 1: `cinder-volume` stopped, then an attach | Volume and attachment `reserved`; Nova lists it; not in `domblklist`; event without result; volume service `down`; a second attach refused with `400 Invalid volume` | `systemctl restart` of the unit, 3.5 min after the request | `in-use`, attachment `attached`, `vdb` in `domblklist`, event `Success`: nothing to force |
| Same, `cinder-volume` kept down | Same, for about 6 min | None needed | Nova gave up and released it: `available`, no attachment, event `Error` |
| Cause 2: `nova-compute` stopped, then a detach | `detaching`; attachment `attached`; still in `domblklist`; compute service `up` for about a minute, then `down` | `systemctl restart` of the unit | `available` within 20 s, event `Success` |
| Cause 4: detach from a paused guest | `detaching` for about 3.5 min, still in `domblklist` | Nova set `in-use` back itself (`DeviceDetachFailed`); guest resumed, detach requested again | `available`, not in `domblklist` |
| `volume attachment delete` of the attachment Nova uses | | | Refused: `409 … ConflictNovaUsingAttachment` |
| Hypervisor-only detach (`virsh detach-disk --live`), both records attached | Not in `domblklist`; Cinder and Nova attached | Hard reboot | Attached again (`vdb` back) |

Not reproduced: cause 3 (no `os_brick` failure was produced). On this all-in-one the compute and the storage are on
the same host.

## Validity and limits

Written from the upstream documentation and client source for 2026.1: the Nova volume administration guide, the
Block Storage API reference (attachment statuses, the HTTP 409 protection), and the OpenStack client's `volume set`
and `volume attachment` commands. Observed on one all-in-one lab with LVM over iSCSI (see Lab observations): causes
1, 2 and 4, their resolutions, the 409 protection and the hard reboot. Not observed: cause 3, other backends (RBD,
multipath), and a multinode platform.

Per release:

- Cinder attachment records (`volume attachment list`, `delete`) need volume API 3.27 or later, available on every
  maintained release.
- The attachment ID in `server volume list` needs compute API 2.89 or later (Xena and later).
- The 409 protection on attachment deletion comes from bug 2004555; on releases or deployments without that fix,
  Cinder accepts the deletion even when Nova still uses the attachment, so check Nova's list yourself first.
- The hard-reboot clean-up of dangling attachments is documented in the 2026.1 Nova guide; check the guide of your
  release before relying on it on an older one.

Not covered: multi-attach volumes, boot volumes (a stuck root volume needs the instance stopped), and host-side
clean-up of leftover iSCSI sessions or multipath devices on the compute.

## Related

- [Stale volume attachment after an instance was deleted or evacuated](stale-volume-attachment-after-instance-deleted-or-evacuated.md)
- [Stale records: stuck instances, orphan ports and attachments, phantom allocations](../lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md)
- Nova, [Manage volumes](https://docs.openstack.org/nova/2026.1/admin/manage-volumes.html)
- Block Storage API, [attachments](https://docs.openstack.org/api-ref/block-storage/v3/#attachments)

---

*Provenance: written by agent:ozarland-author, from upstream documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*
