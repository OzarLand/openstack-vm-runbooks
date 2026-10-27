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

- the command is refused: `Failed to set volume size: --os-volume-api-version 3.42 or greater is required to extend
  in-use volumes.` followed by `One or more of the set operations failed`; or, also from the client before any API
  call, `Volume is in <status> state, it must be available or in-use before size can be extended.` or `New size must
  be greater than <N> GB`; or, from the API, `Volume <volume-id> status must be '...' to extend, currently
  <status>.`;
- the command is accepted, but the volume keeps its old size and status, with a user message `extend volume:Could not
  find any available weighted backend.`;
- the volume goes to `error_extending`;
- Cinder shows the new size, but inside the instance the disk still has the old size.

## Impact

One volume and the instance that uses it. Data is not at risk from the extend itself; a volume in `error_extending`
cannot be extended or detached until its status is corrected (a detached one can still be deleted; an attached one
cannot).

## Quick checks

Read-only. With admin credentials:

```bash
# 1. Cinder's view: size, status, attachment
openstack volume show <volume-id> -c size -c status -c attachments -c os-vol-host-attr:host

# 2. Messages Cinder recorded for the volume (user messages, API 3.3+)
openstack --os-volume-api-version 3.3 volume message list | grep <volume-id>

# 3. The instance's actions: Nova receives a volume-extended event
openstack server event list <server-id>
```

Inside the guest, the disk's size as the guest sees it (`lsblk`).

Logs: the `cinder-volume` service that owns the volume, and the `nova-compute` of the instance's compute:

**Kolla-Ansible:**

```bash
sudo grep -E 'Extend volume|Failed to extend|volume-extended' /var/log/kolla/cinder/cinder-volume.log | tail
sudo grep <volume-id> /var/log/kolla/nova/nova-compute.log | grep -iE 'extend' | tail     # on the compute
sudo grep <volume-id> /var/log/kolla/cinder/cinder-scheduler.log | tail                   # extend not placed
```

The `cinder-volume` lines `Extend volume completed successfully.`, `Extend volume failed.` and `Failed to extend
volume.` do not carry the volume ID (only the `Nova event response` line does, as `tag`): match them by time, or by
the request ID that `volume message list` shows.

**Any:** the same `grep` on the `cinder-volume`, `cinder-scheduler` and `nova-compute` logs.

## Diagnosis

### How an attached volume is extended (Cinder and Nova 2026.1)

1. **Cinder API** (`cinder/volume/api.py`, `_extend`): an attached (`in-use`) volume can be extended only with volume
   API **3.42** or later (`VOLUME_EXTEND_INUSE`); python-openstackclient refuses before calling the API without it.
   The new size must be larger, and within the quota and the volume type's size limits. The status goes to
   `extending`. The request then goes through `cinder-scheduler`, which checks the backend's capacity; if it finds
   none (`Filtering removed all hosts for the request with volume ID '<volume-id>'`, `Failed to schedule_extend_volume`
   in `cinder-scheduler.log`), the volume goes back to its status at its old size, with the user message
   `extend volume:Could not find any available weighted backend.`; the client command itself succeeded.
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
   - If the guest is not running, Nova skips the resize (`Skipping block device resize, guest is not running`, not
     logged at the default INFO level); the guest sees the new size at its next start.

### The guest

Even when the disk is larger, the partition and the file system inside the guest stay as they were: growing them is a
step inside the guest, for its owner.

## Possible causes

### Cause 1: refused by the API or the client

An `in-use` volume without API 3.42; a size not larger than the current one; a quota or volume type limit; a volume
not in `available` or `in-use`; or a policy that forbids extending attached volumes
(`volume:extend_attached_volume`).

### Cause 2: the backend could not grow the volume (`error_extending`)

The `cinder-volume` log has the driver's error: a backend limit, a backend unreachable, a backend volume that
cannot be found. When the driver is not initialised, the volume also goes to `error_extending`, but without a user
message. When the pool has too little space for the new size, the scheduler refuses the extend before the
driver is called: no `error_extending`, the volume keeps its old size and status, and the user message is
`extend volume:Could not find any available weighted backend.` (see the diagnosis).

### Cause 3: Cinder grew it, Nova did not refresh it

The `volume-extended` event did not reach Nova, or the refresh failed on the compute (`Extend volume failed`), or the
instance's `nova-compute` is down. The guest sees the old disk size.

### Cause 4: the disk is larger, the file system is not

`lsblk` in the guest shows the new size, but the partition and file system were not grown.

## Resolution

### For cause 1: use the right API version, or the right size

> ⚠️ **State-changing.** Grows the volume on the backend; a volume cannot be shrunk afterwards. The project's quota is
> charged for the new size. Undo: none (create a smaller volume and copy the data, if needed).

```bash
openstack --os-volume-api-version 3.42 volume set --size <new-GB> <volume-id>
```

### For cause 2: fix the backend, reset the status, extend again

Fix what the `cinder-volume` log names. The volume's size in Cinder is still the old one. Then put the status back to
what it was before the extend:

> ⚠️ **State-changing.** Changes Cinder's record only; the backend is not touched. Precondition: the backend volume
> still has its old size and is healthy (check on the backend), and the volume is attached as Cinder says. Undo:
> `cinder reset-state --state error_extending <volume-id>` (`openstack volume set --state` does not accept
> `error_extending`).

```bash
openstack volume set --state in-use <volume-id>       # or available, if it is not attached
```

Then extend again (cause 1).

### For cause 3: make Nova refresh the device

Bring the instance's `nova-compute` back if it is down, and read the `Extend volume failed` reason. If
`nova-compute` was only down, the event waits in its message queue: once it is back, look for `Cinder extended
volume <volume-id>` in its log and check `lsblk` in the guest before anything else. The refresh is triggered by
Cinder's event, which Cinder does not send again: if the event was lost or the refresh failed, the simplest ways to
make the guest see the new size:

> ⚠️ **State-changing.** A hard reboot stops the guest abruptly: what it had not yet written to disk is lost. A
> detach and attach interrupts its access to that volume (unmount it inside the guest first). Agree it with the
> owner. Undo: none needed.

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

## Lab observations

Kolla-Ansible 22.2.0 (2026.1) all-in-one, Cinder LVM (thin) over iSCSI, python-openstackclient 9.0.0, a CirrOS
instance with a 1 GB data volume, on 2026-10-27:

- Normal path: `Extend volume completed successfully.` 2 s after the command, `Nova event response` 1 s later,
  `Cinder extended volume <volume-id>; extending it to detect new size` in `nova-compute.log` 1 s after that, then
  os-brick's `Extend volume for {...}`; the volume was `in-use` at 2 GB and `lsblk` in the guest showed 2 GB.
- Cause 1: `volume set --size 2` without a microversion printed `Failed to set volume size:
  --os-volume-api-version 3.42 or greater is required to extend in-use volumes.` and `One or more of the set
  operations failed`, exit code 1.
- Extend beyond the thin pool's capacity (500 GB): exit code 0, `CapacityFilter` removed the only backend
  (`Insufficient free virtual space (376.0GB) to accommodate thin provisioned 496GB volume`), the volume stayed
  `in-use` at its old size, user message `extend volume:Could not find any available weighted backend.`
- Cause 2, injected by renaming the backend LV: `error_extending` within 8 s, `Extend volume failed.` (`Failed to
  find logical volume`) and `Failed to extend volume.`, user message `extend volume:Volume Driver failed to extend
  volume.`. After the LV was renamed back, `volume set --state in-use` and the extend again worked.
- Cause 3, `nova-compute` stopped (`systemctl stop kolla-nova_compute-container.service`), then an extend: Cinder
  showed the new size and `in-use` and logged `Nova event response: {'name': 'volume-extended', ... 'status':
  'completed', 'code': 200}`; the guest kept the old size. After `systemctl start`, `nova-compute` processed the
  queued event 18 s later and the guest saw the new size without a reboot. With the event removed from the compute's
  queue while `nova-compute` was down (to simulate a lost event), the guest kept the old size after the start; `server
  reboot --hard` (22 s with `--wait`) brought the new size to the guest.
- Guest shut off during the extend: no `Skipping block device resize` at INFO level; the guest saw the new size at
  its start.

## Validity and limits

- Lab-run on Kolla-Ansible 22.2.0 (2026.1), all-in-one, Cinder LVM over iSCSI (see *Lab observations*): causes 1, 2
  and 3, the capacity refusal and the shut-off guest. Not run: the detach and attach alternative of cause 3, cause 4
  (no file system was grown in the guest), a refresh failure on the compute (`Extend volume failed`), Ceph RBD and
  Fibre Channel.
- Written from Cinder 2026.1 (`cinder/volume/api.py` `_extend`, `cinder/volume/manager.py` `extend_volume` and
  `extend_volume_completion`, `cinder/message/message_field.py`, the 3.42 microversion), Nova 2026.1
  (`nova/compute/manager.py` `extend_volume`, `nova/virt/libvirt/driver.py` `extend_volume`,
  `nova/virt/libvirt/volume/net.py`) and python-openstackclient 2026.1.
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
