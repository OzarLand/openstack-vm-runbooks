---
title: Instance fails to boot from volume. ERROR at build, or ACTIVE with no bootable disk
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, cinder, glance, libvirt]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any multinode
  network: any
  storage: any Cinder backend
  os: any
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-26, note: "full procedure on a disposable lab for causes 1 to 5 (min_disk, no backend, cinder-volume stopped, iscsid stopped, blank bootable volume); reconfigure, iscsi deploy and 2.93 rebuild not run" }
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/user/launch-instance-from-volume.html
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#DEFAULT.block_device_allocate_retries
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#cinder.cross_az_attach
  - https://docs.openstack.org/cinder/2026.1/admin/image-volume-cache.html
  - https://docs.openstack.org/api-ref/block-storage/v3/#messages-messages
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Instance fails to boot from volume. ERROR at build, or ACTIVE with no bootable disk

## Symptom

A user creates an instance whose root disk is a Cinder volume: from an image with a new volume
(`openstack server create --image <image> --boot-from-volume <size-gb> …`), from a volume snapshot, or from an
existing volume (`--volume <volume-id>`). Instances with a local root disk still build. The boot from volume fails
in one of three ways.

**1. Refused at once.** The create command returns an error and no instance is created, for example:

```text
Volume is smaller than the minimum size specified in image metadata. Volume size is <n> bytes, minimum size is <m> bytes.
Block Device Mapping is Invalid: Boot sequence for the instance and image/block device mapping combination is not valid.
The availability zone between the server and its attached volumes do not match: <reason>.
```

**2. ERROR during the build.** The instance goes to `ERROR`, after a few seconds or after about three minutes, and
its fault names the volume Nova was waiting for (observed on Nova 2026.1):

```text
$ openstack server show <server-id> -c status -c fault
| fault  | {'code': 500, 'message': 'Build of instance <server-id> aborted: Volume <volume-id> did not finish being created even after we waited 185 seconds or 61 attempts. And its status is creating.', ...} |
| status | ERROR                                                                                                  |
```

The status at the end is `error` when Cinder failed (then after a few seconds: `waited 0 seconds or 1 attempts`),
`creating` or `downloading` when Cinder was still working after about three minutes. Older releases or other failures
of the block device step can show `Failure prepping block device.` instead.

The reason after "aborted:" can also be `Block Device Mapping is Invalid: failed to get volume <volume-id>.` (or
`snapshot`, or `image`), or a quota message from Cinder. When the volume was ready but the compute could not connect
to it, the build is rescheduled to other computes, and the fault ends as `Exceeded maximum number of retries.`

**3. ACTIVE, but nothing boots.** The instance is `ACTIVE`, does not answer, and its console shows no operating
system starting:

```text
$ openstack console log show <server-id>
(empty, or a firmware message such as "No bootable device")
```

## Impact

Every new instance that boots from a volume, for one user or for the whole platform, depending on the cause.
Running instances are not affected. No data is at risk from the failure itself: a failed build leaves at most an
unused volume behind. Two steps of cause 5 can delete or overwrite a boot volume; they are marked 🛑. The risk
comes from retrying blindly: each retry from an image creates another volume, which counts against the project's
Cinder quota and fills the backend.

## Quick checks

Read-only. With admin credentials, from any host with the OpenStack client:

```bash
# 1. The instance's status and fault (the fault only appears on an ERROR instance)
openstack server show <server-id> -c status -c fault -c OS-EXT-SRV-ATTR:host -c created

# 2. The volumes of the build: their status, size, bootable flag and availability zone
openstack server volume list <server-id>
openstack volume list --all-projects --long | grep -i -e error -e creating -e downloading
openstack volume show <volume-id> -c status -c size -c bootable -c availability_zone -c volume_image_metadata

# 3. Cinder's own explanation for a volume in error (user messages need volume API 3.3 or later)
openstack --os-volume-api-version 3.3 volume message list | grep <volume-id>

# 4. Are the volume services up?
openstack volume service list

# 5. The image's format and minimum size, when booting from an image
openstack image show <image-id> -c disk_format -c min_disk -c size
```

How to read them:

| You see | Go to |
|---|---|
| The create command was refused, no instance | [Cause 1](#cause-1-the-request-is-refused-by-nova) |
| Fault `… did not finish being created … And its status is error.` (or `Failure prepping block device.`); the volume is in `error`, or was deleted; check 3 may show a Cinder message | [Cause 2](#cause-2-cinder-could-not-create-the-volume) |
| Fault `… did not finish being created even after we waited <n> seconds or 61 attempts. And its status is creating.` after about three minutes; the volume is or was `creating` or `downloading` | [Cause 3](#cause-3-the-volume-was-not-ready-in-time) |
| Fault `Exceeded maximum number of retries.`; the volume is `available` | [Cause 4](#cause-4-the-compute-cannot-connect-to-the-volume) |
| `ACTIVE`, console empty or "No bootable device" | [Cause 5](#cause-5-the-volume-holds-no-bootable-system) |

## Diagnosis

### Nova

The build of a volume-backed instance goes through a block device step before the instance is spawned
(`task_state` `block_device_mapping`). For a new volume (from an image, a snapshot or blank), `nova-compute` asks
Cinder to create it, then checks its status every `block_device_allocate_retries_interval` seconds (3 by default),
up to `block_device_allocate_retries` + 1 times (61 by default): about three minutes. A volume that is still
`creating` or `downloading` after that, or that ends in any status other than `available`, aborts the build with
`Volume <volume-id> did not finish being created …` as the fault, and Nova does not try another compute. The detailed error is only in the
`nova-compute` log of the compute that tried:

**Kolla-Ansible:** `/var/log/kolla/nova/nova-compute.log` on the compute named by `OS-EXT-SRV-ATTR:host` (empty on
an ERROR instance: search every compute's log for the server ID).

**Any:** the `nova-compute` service log on the compute.

What to search for, with the server ID:

- `Volume <volume-id> did not finish being created even after we waited <n> seconds or <n> attempts. And its status is <status>.`:
  the status at the end says whether Cinder was still working (`creating`, `downloading`) or had failed (`error`);
- `Instance failed block device setup: nova.exception.VolumeNotCreated` (on a lab, `Failure prepping block device`
  did not appear in the log for causes 2 and 3);
- `Instance failed to spawn` with an `os_brick` exception: the volume existed, but the compute could not connect to
  it (cause 4); Nova then reschedules the build to another compute.

### Cinder

For a volume in `error`, check 3 often gives the reason directly, for example `Could not find any available
weighted backend.` (scheduler), `Not enough quota resource for operation.`, `Image used for creating volume
exceeds available space.`, or `Driver failed to create the volume.` Then read the logs, searching for the volume ID:

**Kolla-Ansible:** `/var/log/kolla/cinder/cinder-scheduler.log` and `cinder-volume.log`, on the hosts that run
`cinder_scheduler` and `cinder_volume` (the `cinder` group, which defaults to the control hosts in 2026.1).

**Any:** the `cinder-scheduler` and `cinder-volume` service logs.

For a volume created from an image, `cinder-volume` downloads the image from Glance and writes it to the volume,
converting it to raw first when its format is not raw, unless the backend can clone the image directly (for
example Ceph RBD with Glance images in the same cluster). The log shows how long that took.

### Glance

When the volume is created from an image, check that the image is `active`, that its `disk_format` matches its
content (an image declared `raw` that is in fact qcow2 is written to the volume as it is, and does not boot), and
that its `min_disk` is not larger than the requested volume size.

## Possible causes

### Cause 1: the request is refused by Nova

Nova checks the request before creating anything. Common refusals:

- the volume is smaller than the image's `min_disk` (`Volume is smaller than the minimum size specified in image
  metadata`);
- `--volume <volume-id>` names a volume whose `bootable` flag is `false`, for example a blank volume
  (`Block Device <volume-id> is not bootable.`, HTTP 400);
- no block device has boot index 0 (`Boot sequence for the instance and image/block device mapping combination is
  not valid`);
- `[cinder] cross_az_attach = False` in Nova and the volume is in another Cinder availability zone than the
  instance (`The availability zone between the server and its attached volumes do not match`). The default is
  `True`, so this only happens where it was changed;
- the flavor has `disk = 0` and the request has no volume: `Only volume-backed servers are allowed for flavors with
  zero disk.` That one is the opposite case, a boot that is not from a volume.

### Cause 2: Cinder could not create the volume

The volume went to `error`. Nova saw it at its next check and aborted the build within seconds. Cinder's message
and logs say why: no backend with enough free space or matching the volume type (scheduler), the project's volume
or gigabytes quota, no space to download or convert the image (`image_conversion_dir` on the `cinder-volume`
host), a backend driver error, or the `cinder-volume` service down. With `[cinder] cross_az_attach = False`, Nova
also creates the volume in the instance's availability zone: if Cinder has no zone of that name, it refuses the
volume.

### Cause 3: the volume was not ready in time

Cinder was still creating the volume (`creating`, or `downloading` the image) when Nova's checks ran out. Large
images, raw images of several gigabytes, a slow Glance backend or a busy `cinder-volume` service take longer than
the three minutes Nova waits by default. Cinder may finish later: the volume then ends `available` and unused, or
is deleted by Nova's clean-up if `delete_on_termination` was set.

### Cause 4: the compute cannot connect to the volume

The volume is `available`, but `nova-compute` fails to attach it when spawning the instance: `os_brick` cannot log in
to the iSCSI target, find the multipath device, or open the RBD image. Typical reasons: the `iscsid` service missing
or stopped on the compute (Kolla-Ansible deploys `iscsid` on computes when an iSCSI backend such as LVM is enabled),
the storage network not reachable from the compute, or, for Ceph, the client key or libvirt secret missing or wrong on
the compute. Nova reschedules the build to other computes; when they all fail the same way, the fault is
`Exceeded maximum number of retries.`

### Cause 5: the volume holds no bootable system

The instance is `ACTIVE` because Nova attached the volume and started the guest, but there is nothing to boot:

- the volume holds no operating system but is marked bootable (for example a blank volume after
  `openstack volume set --bootable`; with `bootable` `false`, Nova refuses the request, see cause 1). On a lab the
  instance was `ACTIVE` and `console log show` was empty;
- the image's `disk_format` does not match its content, so Cinder wrote it as it is. Cinder 2026.1 checks this and
  refuses a qcow2 file declared `raw`: the volume goes to `error`
  (`ImageUnacceptable: … The image format was claimed to be 'raw' but the image data appears to be in a different
  format.` in `cinder-volume.log`, no user message), which is cause 2, not this one;
- the image needs firmware or a disk bus the instance does not get (for example a UEFI-only image without
  `hw_firmware_type=uefi`, or a driver missing for the disk bus), which shows as a guest that starts then cannot
  find its root disk.

## Resolution

Work through the checks by risk: the read-only quick checks first, then the state-changing steps below, the
smallest first.

### For cause 1

Read-only: fix the request, not the platform. Ask for a volume at least as large as the image's `min_disk`, give
the boot volume boot index 0 (`--boot-from-volume` or `--volume` does), and use a volume in the instance's
availability zone if `cross_az_attach` is `False`.

### For cause 2

Fix what Cinder reported, then retry the build once:

- quota: `openstack quota show <project-id>` (read-only) shows the volume and gigabytes limits; raising them is a
  decision for whoever owns the project's quota;
- no backend: `openstack volume service list` and the scheduler log show which backend was rejected and why (free
  space, volume type, a backend `down`);
- a full LVM volume group (LVM backend): on the hosts of the `cinder-volume-lvm` group (or the host running
  `cinder-volume` for LVM), `sudo vgs cinder-volumes` shows the free space left (`VFree`); `cinder-volumes` is
  Kolla-Ansible's default `cinder_volume_group`;
- `cinder-volume` down: see why in its log before restarting it.

> ⚠️ **State-changing.** Restarts the `cinder-volume` service of this host. Volumes being created or attached
> through it at that moment can fail and need retrying; running instances keep their attached volumes. Check that
> the log shows why it stopped first: a restart does not fix a full disk or an unreachable backend.

**Kolla-Ansible:**

```bash
sudo systemctl restart kolla-cinder_volume-container.service
```

**Any:**

```bash
sudo systemctl restart <cinder-volume-unit>
```

### For cause 3

First, make the image faster to turn into a volume, without changing Nova:

- if the volume ended `available` after the build failed, boot from that volume instead of creating a new one:
  `openstack server create --volume <volume-id> --flavor <flavor> --network <network> <name>`;
- enable Cinder's image-volume cache on the backend, which the Nova option's own help recommends, so that later
  volumes from the same image are cloned instead of downloaded ([image-volume cache](https://docs.openstack.org/cinder/2026.1/admin/image-volume-cache.html));
- with Ceph RBD for both Glance and Cinder, upload images as `raw` so that Cinder can clone them instead of
  downloading and converting them.

If volumes are created correctly but always take longer than three minutes, let Nova wait longer on the computes.

> ⚠️ **State-changing.** `reconfigure` restarts the `nova_compute` containers whose configuration changed; running
> instances are not affected. Undo: remove the override and reconfigure again. A longer wait also delays the
> `ERROR` of builds that will never succeed.

**Kolla-Ansible:**

```ini
# /etc/kolla/config/nova/nova-compute.conf
[DEFAULT]
block_device_allocate_retries = 180
```

```bash
# not run on the lab
kolla-ansible reconfigure -i <inventory> --tags nova
```

**Any:** *(not run on the lab)* set `block_device_allocate_retries` in the `[DEFAULT]` section of `nova.conf` on the computes and restart
`nova-compute`. With the default 3-second interval, 180 retries is about nine minutes.

### For cause 4

Read-only first: on a compute where the build failed, check that the compute reaches the storage. For an iSCSI
backend, `iscsid` must run on the compute (`sudo docker ps -a --filter name=iscsid` on Kolla-Ansible; without
`-a` a stopped container is not listed); for Ceph, the `nova-compute` log names the RBD error. Fix the host or its
configuration (storage network, client key), then retry the build once. On Kolla-Ansible, a stopped `iscsid`
container is started with `sudo systemctl start kolla-iscsid-container.service`.

On Kolla-Ansible, if no `iscsid` container exists at all, it is not a container to start by hand: Kolla-Ansible
deploys `iscsid` (`enable_iscsid`) only when an iSCSI backend is enabled (`enable_cinder_backend_iscsi`, which
follows `enable_cinder_backend_lvm`). A backend that uses iSCSI but was configured only through overrides needs
`enable_cinder_backend_iscsi: true` in `globals.yml`, then a deploy, so that `iscsid` lands on the hosts of the
`iscsid` group (compute, storage, ironic).

> ⚠️ **State-changing.** Deploys the services that the new `globals.yml` value enables and can restart
> containers whose configuration changes. Run it in a maintenance window; running instances are not affected by
> the new `iscsid` containers. Undo: set the value back and deploy again (the containers stay until removed).

```bash
# not run on the lab
kolla-ansible deploy -i <inventory> --tags iscsi
```

The [volume stuck in attaching](../volumes/volume-stuck-in-attaching-or-detaching.md) runbook covers `os_brick`
errors in more detail.

### For cause 5

Read-only: check the volume and the image (`bootable`, `volume_image_metadata`, `disk_format`, the firmware and bus
properties).

Fixing a property and rebooting does not help: Nova uses the image properties it copied into the instance at build
time, so the running instance keeps the old ones. The fix is a new build from a corrected source:

- an image with the wrong `disk_format`: upload it again with the right format (check with `qemu-img info` on the
  file), then boot from a new volume created from it;
- a blank volume: boot from a volume created from an image or a snapshot that holds an operating system;
- a missing property (firmware, disk bus): set it on the volume, or on the image for every later volume, then
  build a new instance from that volume.

> ⚠️ **State-changing.** Changes the image metadata stored on the volume (or on the image); the data on the volume
> does not change. Undo: `openstack volume unset --image-property <key> <volume-id>`.

```bash
# not run on the lab
openstack volume set --image-property hw_firmware_type=uefi <volume-id>      # example property
openstack image set --property hw_firmware_type=uefi <image-id>              # for later volumes from this image
```

To build from the same volume, first free it from the failed instance. Check whether deleting the instance would
delete the volume (the column needs compute API 2.79 or later):

```bash
openstack --os-compute-api-version 2.79 server volume list <server-id>        # read-only: "Delete On Termination"
```

> 🛑 **Destructive.** Deletes the failed instance. With "Delete On Termination" `False`, the volume is kept and
> becomes `available`; with `True`, **the volume is deleted with the instance**, with everything on it. Check that
> column first; if it is `True`, do not delete the instance to reuse the volume: build from a new volume instead.

```bash
openstack server delete --wait <server-id>
openstack server create --volume <volume-id> --flavor <flavor> --network <network> <name>
```

Where the compute API is 2.93 or later (Zed and later, with Cinder supporting volume reimage, block storage API
3.68), a volume-backed instance can also be rebuilt from an image that carries the right properties, keeping the
instance:

> 🛑 **Destructive.** Rebuild rewrites the instance's boot volume from the image: everything on that volume is
> replaced. Only for an instance whose volume holds nothing to keep, such as one that never booted.

```bash
# not run on the lab
openstack --os-compute-api-version 2.93 server rebuild --image <image-id> <server-id>
```

## Verification

```bash
# The new instance is ACTIVE, its boot volume in-use and bootable
openstack server show <server-id> -c status -c fault
openstack server volume list <server-id>
openstack volume show <volume-id> -c status -c bootable

# The guest starts its operating system
openstack console log show <server-id> | tail -20

# No volume left behind by the failed attempts
openstack volume list --project <project-id> --status available --long
```

Volumes left `available` by failed builds are not deleted by this runbook: either reuse them (cause 3), or let their
owner decide whether to delete them.

## Lab observations

On a lab: OpenStack 2026.1 (Nova 33.0.3), Kolla-Ansible 22.2.0, all-in-one, Cinder LVM (thin) over iSCSI,
python-openstackclient 9.0.0, `--boot-from-volume` from a cirros image.

- **Cause 1.** An image with `min_disk` 2 and `--boot-from-volume 1`: refused at once with `Volume is smaller than the
  minimum size specified in image metadata. Volume size is 1073741824 bytes, minimum size is 2147483648 bytes.` A
  blank volume with `--volume`: refused with `Block Device <volume-id> is not bootable.`
- **Cause 2.** `--boot-from-volume 500` on a pool with 380 GB of virtual capacity: `ERROR` about 13 s after the request,
  fault `… did not finish being created even after we waited 0 seconds or 1 attempts. And its status is error.`;
  check 3 gave `schedule allocate volume:Could not find any available weighted backend.`; `cinder-scheduler.log`
  named `CapacityFilter` (`Insufficient free virtual space (380.0GB) to accommodate thin provisioned 500GB volume`).
  The volume stayed in `error` after the instance was deleted. `server volume list` of the ERROR instance was empty.
- **Cause 3.** `cinder-volume` stopped just before the create: the instance was `ERROR` 3 min 16 s after the request
  (`waited 185 seconds or 61 attempts. And its status is creating.`), the volume `creating` with its host set. When
  `cinder-volume` was started again, it took the queued request and created the volume (`available` 27 s after
  the restart), not `error`. Booting from that volume with `--volume` worked.
- **Cause 4.** `iscsid` container stopped: `ERROR` about 40 s after the request with `Exceeded maximum number of retries.
  Exhausted all hosts available for retrying build failures`; `nova-compute.log` showed `Failed to login iSCSI target
  … (exit code 20)` then `Instance failed to spawn: os_brick.exception.VolumeDeviceNotFound: Volume device not found
  at .` The volume was left `available` and `bootable`; with `iscsid` started, `--volume` from it booted.
- **Cause 5.** A blank volume set `--bootable`: `ACTIVE`, console log empty. Deleting the instance (Delete On
  Termination `False`) left the volume `available`. A qcow2 image uploaded as `raw`: Glance accepted it, Cinder
  refused it (see cause 5).

Not run: the `block_device_allocate_retries` reconfigure, the `iscsi` deploy, the image or volume property changes and
the 2.93 rebuild of cause 5, `cross_az_attach`, Ceph.

## Validity and limits

- Written from Nova 2026.1 (`nova/compute/manager.py`, `nova/virt/block_device.py`, `nova/exception.py`, the
  `block_device_allocate_retries` and `cross_az_attach` options), Cinder 2026.1 (user messages in `cinder/message/message_field.py`) and
  Kolla-Ansible 2026.1. Causes 1 to 5 partly reproduced on an all-in-one lab (see Lab observations).
- The fault messages are quoted from Nova's and Cinder's source; their exact form in `openstack server show` may vary
  with the client and the compute API microversion.
- Not covered: Ironic bare metal instances, rescue and rebuild of volume-backed instances, encrypted volumes,
  multi-attach volumes, volume types with a QoS or replication policy, and boot from a volume of another project.
- The causes are the common ones the source makes visible, not a complete list.

## Related

- [No valid host was found, although capacity exists](no-valid-host-with-free-capacity.md): when the build fails
  before reaching a compute.
- [Volume stuck in attaching, reserved or detaching](../volumes/volume-stuck-in-attaching-or-detaching.md)
- [Stale records: stuck instances, orphan ports and attachments](../lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md)
- Nova: [launch an instance from a volume](https://docs.openstack.org/nova/2026.1/user/launch-instance-from-volume.html),
  [`block_device_allocate_retries`](https://docs.openstack.org/nova/2026.1/configuration/config.html#DEFAULT.block_device_allocate_retries),
  [`cross_az_attach`](https://docs.openstack.org/nova/2026.1/configuration/config.html#cinder.cross_az_attach)
- Cinder: [image-volume cache](https://docs.openstack.org/cinder/2026.1/admin/image-volume-cache.html),
  [messages API](https://docs.openstack.org/api-ref/block-storage/v3/#messages-messages)

---

*Provenance: written by agent:ozarland-author, from the Nova, Cinder and Kolla-Ansible 2026.1 documentation and
source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=scheduling/instance-fails-to-boot-from-volume.md&title=Feedback%3A+scheduling/instance-fails-to-boot-from-volume.md):
your release, your deployment method, and what happened.
