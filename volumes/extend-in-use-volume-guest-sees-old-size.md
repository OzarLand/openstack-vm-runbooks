---
title: Extending an attached volume. Refused, error_extending, or the instance still sees the old size
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [cinder, nova, libvirt, os-brick]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: any
  storage: any Cinder backend; Ceph RBD, iSCSI and Fibre Channel named where they differ
  os: any
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/cinder/2026.1/contributor/api_microversion_history.html
  - https://docs.openstack.org/cinder/2026.1/admin/manage-volumes.html
  - https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/volume.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Extending an attached volume. Refused, error_extending, or the instance still sees the old size

## Symptom

One of these, after `openstack volume set --size <new-GB> <volume-id>` on a volume attached to an instance:

- the command is refused: `--os-volume-api-version 3.42 or greater is required to extend in-use volumes.`, or
  `Volume <volume-id> status must be '...' to extend, currently <status>.`;
- the volume goes to `error_extending`;
- Cinder shows the new size, but inside the instance the disk still has the old size.

## Impact

One volume and the instance that uses it. Data is not at risk from the extend itself; a volume in `error_extending`
cannot be extended, detached or deleted until its status is corrected.

## Quick checks

Read-only. With admin credentials:

```bash
# 1. Cinder's view: size, status, attachment
openstack volume show <volume-id> -c size -c status -c attachments -c os-vol-host-attr:host

# 2. Messages Cinder recorded for the volume (user messages, API 3.3+)
openstack --os-volume-api-version 3.3 volume message list --resource-uuid <volume-id>

# 3. The instance's actions: Nova receives a volume-extended event
openstack server event list <server-id>
```

Inside the guest, the disk's size as the guest sees it (`lsblk`).

Logs: the `cinder-volume` service that owns the volume, and the `nova-compute` of the instance's compute:

**Kolla-Ansible:**

```bash
sudo grep <volume-id> /var/log/kolla/cinder/cinder-volume.log | grep -iE 'extend' | tail
sudo grep <volume-id> /var/log/kolla/nova/nova-compute.log | grep -iE 'extend' | tail     # on the compute
```

**Any:** the same `grep` on the `cinder-volume` and `nova-compute` logs.

## Diagnosis

### How an attached volume is extended (Cinder and Nova 2026.1)

1. **Cinder API** (`cinder/volume/api.py`, `_extend`): an attached (`in-use`) volume can be extended only with volume
   API **3.42** or later (`VOLUME_EXTEND_INUSE`); python-openstackclient refuses before calling the API without it.
   The new size must be larger, and within the quota and the volume type's size limits. The status goes to
   `extending`.
2. **`cinder-volume`** (`cinder/volume/manager.py`, `extend_volume`) asks the backend driver to grow the volume:
   - on failure: `Extend volume failed.` and `Failed to extend volume.` in the log, a user message `Volume Driver
     failed to extend volume.`, and the status `error_extending`;
   - on success: the size is updated, the status goes back to `in-use`, `Extend volume completed successfully.` is
     logged, and Cinder sends Nova a `volume-extended` event for each instance the volume is attached to.
3. **`nova-compute`** (`nova/compute/manager.py`, `extend_volume`) logs `Cinder extended volume <id>; extending it to
   detect new size`, then asks the libvirt driver to refresh the device: for iSCSI and Fibre Channel, os-brick rescans
   the host's device; for network volumes such as Ceph RBD there is no host device to rescan. Then QEMU is told the
   new size, and the guest sees a larger disk.
   - On failure, `nova-compute` logs `Extend volume failed, volume_id=<id>, reason: <error>`. **Cinder is not told**:
     it shows the new size and `in-use`, while the guest still sees the old size.
   - If the guest is not running, Nova skips the resize (`Skipping block device resize, guest is not running`); the
     guest sees the new size at its next start.

### The guest

Even when the disk is larger, the partition and the file system inside the guest stay as they were: growing them is a
step inside the guest, for its owner.

## Possible causes

### Cause 1: refused by the API or the client

An `in-use` volume without API 3.42; a size not larger than the current one; a quota or volume type limit; a volume
not in `available` or `in-use`.

### Cause 2: the backend could not grow the volume (`error_extending`)

The `cinder-volume` log has the driver's error: no space in the pool, a backend limit, a backend unreachable.

### Cause 3: Cinder grew it, Nova did not refresh it

The `volume-extended` event did not reach Nova, or the refresh failed on the compute (`Extend volume failed`), or the
instance's `nova-compute` is down. The guest sees the old disk size.

### Cause 4: the disk is larger, the file system is not

`lsblk` in the guest shows the new size, but the partition and file system were not grown.

## Resolution

### For cause 1: use the right API version, or the right size

```bash
openstack --os-volume-api-version 3.42 volume set --size <new-GB> <volume-id>
```

> ⚠️ **State-changing.** Grows the volume on the backend; a volume cannot be shrunk afterwards. The project's quota is
> charged for the new size. Undo: none (create a smaller volume and copy the data, if needed).

### For cause 2: fix the backend, reset the status, extend again

Fix what the `cinder-volume` log names. The volume's size in Cinder is still the old one. Then put the status back to
what it was before the extend:

> ⚠️ **State-changing.** Changes Cinder's record only; the backend is not touched. Precondition: the backend volume
> still has its old size and is healthy (check on the backend), and the volume is attached as Cinder says. Undo: set
> `error_extending` back.

```bash
openstack volume set --state in-use <volume-id>       # or available, if it is not attached
```

Then extend again (cause 1).

### For cause 3: make Nova refresh the device

Bring the instance's `nova-compute` back if it is down, and read the `Extend volume failed` reason. The refresh is
triggered by Cinder's event, which is not sent again. The simplest ways to make the guest see the new size:

> ⚠️ **State-changing.** A hard reboot stops the guest abruptly; a detach and attach interrupts its access to that
> volume (unmount it inside the guest first). Agree it with the owner. Undo: none needed.

```bash
openstack server reboot --hard <server-id>              # the volume is reattached at its new size
```

or, for a data volume, detach and attach it again (`openstack server remove volume` then `openstack server add
volume`).

### For cause 4: grow the partition and the file system in the guest

The owner does it inside the guest, with the guest's tools (for example `growpart` and `resize2fs` or `xfs_growfs`
on Linux).

## Verification

```bash
openstack volume show <volume-id> -c size -c status     # the new size, in-use
```

Inside the guest: `lsblk` shows the new size, `df -h` the grown file system.

## Validity and limits

- Written from Cinder 2026.1 (`cinder/volume/api.py` `_extend`, `cinder/volume/manager.py` `extend_volume` and
  `extend_volume_completion`, `cinder/message/message_field.py`, the 3.42 microversion), Nova 2026.1
  (`nova/compute/manager.py` `extend_volume`, `nova/virt/libvirt/driver.py` `extend_volume`,
  `nova/virt/libvirt/volume/net.py`) and python-openstackclient 2026.1. Not reproduced on a lab.
- Not covered: shrinking, encrypted volumes (Nova handles them differently), multi-attached volumes, and extending a
  root volume of a running instance from the guest's point of view.

## Related

- [Volume stuck in attaching or detaching](volume-stuck-in-attaching-or-detaching.md)
- Cinder 2026.1: [API microversion history](https://docs.openstack.org/cinder/2026.1/contributor/api_microversion_history.html)

---

*Provenance: written by agent:ozarland-author, from the Cinder, Nova and python-openstackclient 2026.1 source and
documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=volumes/extend-in-use-volume-guest-sees-old-size.md&title=Feedback%3A+volumes/extend-in-use-volume-guest-sees-old-size.md):
your release, your deployment method, and what happened.
