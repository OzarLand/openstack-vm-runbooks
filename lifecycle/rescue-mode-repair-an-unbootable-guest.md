---
title: The guest does not boot or cannot be reached (broken fstab, kernel, network or SSH configuration). Repair its disk in rescue mode
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [nova, libvirt, glance]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: any
  storage: local instance disks; volume-backed instances with the stable device rescue
  os: any
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/user/rescue.html
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#DEFAULT.rescue_timeout
  - https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/server.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# The guest does not boot or cannot be reached (broken fstab, kernel, network or SSH configuration). Repair its disk in rescue mode

## Symptom

Nova sees the instance as `ACTIVE`, but the guest operating system is broken. Its console log or console shows that it
stops at boot (a kernel panic, an emergency shell after a wrong `/etc/fstab` line, a full root file system), or it
boots but cannot be reached: a broken network configuration, a firewall rule, a lost SSH key or password. A reboot
does not help: the fault is inside the guest's disk.

```bash
openstack console log show --lines 50 <server-id>      # the end of the guest's boot messages
```

## Impact

One instance, down or unreachable for its users. Its data is intact on its disk. Rescue mode stops the guest while
the repair is done.

## Quick checks

Read-only:

```bash
# 1. State, image, and whether it boots from a volume (an empty image field, or volumes_attached)
openstack server show <server-id> -c status -c image -c volumes_attached -c OS-EXT-SRV-ATTR:host

# 2. What the guest prints at boot
openstack console log show --lines 100 <server-id>
```

The console (`openstack console url show <server-id>`) shows an emergency shell or a login prompt when the console
log is empty.

## Diagnosis

### What rescue does (Nova 2026.1)

`openstack server rescue` stops the guest (a clean shutdown first, within `shutdown_timeout`), then starts it from a
**rescue disk**, with the instance's own disks attached to it as data disks. The instance goes to `RESCUE`. From the
rescue system, the operator mounts the original root disk and repairs it. `openstack server unrescue` stops the rescue
system and boots the instance from its own disk again (Nova user guide, *Rescue an instance*). The API accepts a
rescue only for an instance in `ACTIVE`, `SHUTOFF` or `ERROR`; any other state returns HTTP 409.

Two modes (`nova/virt/libvirt/blockinfo.py`):

- **Legacy rescue** (default): the rescue disk is the first disk, the original root disk comes second (`vdb` on a
  `virtio` bus), then the config drive if any. The rescue disk is a fresh copy of the rescue image: the image given
  with `--image`, otherwise `[libvirt] rescue_image_id` if set, otherwise the instance's own image, which must then
  still exist in Glance. Not possible for a volume-backed instance (`Cannot rescue a
  volume-backed instance`).
- **Stable device rescue** (since Ussuri, libvirt only, not LXC): the libvirt driver uses it as soon as the rescue
  image has `hw_rescue_device` **or** `hw_rescue_bus` (`check_hw_rescue_props` in `nova/virt/hardware.py`), with or
  without API 2.87, and logs `Attempting a stable device rescue`. Every original device stays where it was, so the
  original root disk stays **first**, and the rescue disk is added at the end. It is the only way to rescue a
  **volume-backed** instance, which needs compute API 2.87 or later, both properties on the image (the API checks
  both for that case), and a compute that reports the `COMPUTE_RESCUE_BFV` trait.

While in `RESCUE`, the upstream guide says not to pause, suspend or stop the instance: that loses the original
instance state and makes unrescue impossible.

### Messages

| Message | Meaning |
|---|---|
| `Instance <id> cannot be rescued: Cannot rescue a volume-backed instance` | Volume-backed, and no stable device rescue requested: use an image with `hw_rescue_*` and API 2.87 |
| `Instance <id> cannot be rescued: Host unable to rescue a volume-backed instance` | The compute does not report `COMPUTE_RESCUE_BFV` |
| `Requested rescue image '<image>' is not supported` | The image does not exist, or is a volume snapshot image (`img_block_device_mapping`) |
| `Instance <id> cannot be rescued: Driver Error: <error>` | libvirt failed to start the rescue system; the `nova-compute` log has `Error trying to Rescue Instance` with the traceback |

## Possible causes

The causes are inside the guest; this page is about getting to its disk. The usual ones: a wrong `/etc/fstab` entry, a
kernel or initramfs update that does not boot, a full root file system, a network or firewall configuration that cuts
the guest off, an SSH key or password lost.

## Resolution

### 1. Rescue the instance

Check `[DEFAULT] rescue_timeout` on the instance's compute first (default 0, off): if it is set, Nova unrescues the
instance without warning when it expires, possibly while you have its disk mounted.

Choose the rescue image: the instance's own image works for most Linux guests; a dedicated rescue image is safer when
the guest finds its root file system by label or UUID: with the instance's own image, the rescue disk is a copy of
the same image, so both disks carry the same label and UUID, and the rescue system may mount the wrong one.

> ⚠️ **State-changing.** Stops the guest (clean shutdown first) and boots the rescue system instead; the instance's
> users lose it until the unrescue. Its disks are not changed by the rescue itself. Undo:
> `openstack server unrescue <server-id>`.

```bash
openstack server rescue <server-id>                                  # legacy rescue, the instance's image
openstack server rescue --image <rescue-image> <server-id>          # another image
openstack server rescue --password <password> <server-id>          # a password for the rescue system, if the driver
                                                                     # and image support setting it
openstack --os-compute-api-version 2.87 server rescue --image <image-with-hw_rescue-properties> <server-id>
                                                                     # volume-backed instance (stable device rescue)
openstack server show <server-id> -c status                         # RESCUE
```

Log in to the rescue system through the console, or by SSH with the instance's key pair if the rescue image accepts
it.

### 2. Repair the original disk

From the rescue system, find the original root disk with `lsblk` and `blkid`: with legacy rescue it is the second
disk; with stable device rescue (an image with `hw_rescue_*` properties) it keeps its place, usually the first, and the
rescue disk is the last. Check its size and file systems before mounting it, and
fix the cause: the `fstab` line, the boot entry, free space, the network or SSH configuration.

> ⚠️ **State-changing.** Changes files on the instance's own disk. Precondition: the owner agreed to the change and
> knows what is edited. Keep a copy of every file you change (next to it, with a suffix), so it can be put back.

Unmount the disk before leaving the rescue system.

### 3. Unrescue

> ⚠️ **State-changing.** Stops the rescue system and boots the instance from its own disk. Undo: rescue again.

```bash
openstack server unrescue <server-id>
openstack server show <server-id> -c status        # ACTIVE
```

`[DEFAULT] rescue_timeout` (default 0, off) makes `nova-compute` unrescue instances that stay in `RESCUE` longer than
that many seconds, without warning; if it is set, finish the repair and unmount within it.

## Verification

```bash
openstack server show <server-id> -c status
openstack console log show --lines 50 <server-id>      # a normal boot, ending at the login prompt
```

Then reach the instance as its users do (SSH, its service port).

## Validity and limits

- Written from Nova 2026.1 (*Rescue an instance* user guide, `nova/compute/api.py` `rescue` and `unrescue`,
  `nova/compute/manager.py` `rescue_instance` and `_poll_rescued_instances`, `nova/virt/libvirt/blockinfo.py`,
  `nova/exception.py`, `rescue_timeout`) and python-openstackclient 2026.1. Not reproduced on a lab.
- The repair inside the guest depends on its operating system; this page does not cover Windows guests, encrypted
  root file systems, or LVM inside the guest.
- Not covered: drivers other than libvirt, and rescuing an instance in `ERROR` whose disk is not on its host (see
  [resize not finished](resize-stuck-in-verify-resize-or-failed.md)).

## Related

- [New instance has an address, but no SSH key](../networking/new-instance-boots-without-metadata.md): when the key was
  never injected, rather than lost inside the guest.
- Nova 2026.1: [rescue an instance](https://docs.openstack.org/nova/2026.1/user/rescue.html)

---

*Provenance: written by agent:ozarland-author, from the Nova and python-openstackclient 2026.1 documentation and
source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=lifecycle/rescue-mode-repair-an-unbootable-guest.md&title=Feedback%3A+lifecycle/rescue-mode-repair-an-unbootable-guest.md):
your release, your deployment method, and what happened.
