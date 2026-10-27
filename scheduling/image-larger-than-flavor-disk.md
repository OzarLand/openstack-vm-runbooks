---
title: Boot refused or aborted because the flavor's disk is too small for the image
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, glance, cinder]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: any
  storage: local instance disks or Ceph RBD; boot from volume where named
  os: any
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-27, note: "full procedure on a disposable lab with local disks: cause 2 (virtual size above the flavor disk, build aborted on the compute), cause 3 (zero-disk flavor, admin and member), cause 4 (min_ram), min_disk and boot-from-volume resolutions; not run: the API refusal on the image file size, a Ceph RBD image backend" }
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/user/flavors.html
  - https://docs.openstack.org/glance/2026.1/admin/useful-image-properties.html
  - https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/image-v2.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Boot refused or aborted because the flavor's disk is too small for the image

## Symptom

`openstack server create` is refused at once (HTTP 400), or the instance goes to `ERROR` a little later, with one of
these messages (`nova/exception.py`):

```text
Flavor's disk is too small for requested image. Flavor disk is <n> bytes, image is <n> bytes.
Flavor's disk is smaller than the minimum size specified in image metadata. Flavor disk is <n> bytes, minimum size is <n> bytes.
Volume is smaller than the minimum size specified in image metadata. Volume size is <n> bytes, minimum size is <n> bytes.
Flavor's memory is too small for requested image.
Only volume-backed servers are allowed for flavors with zero disk.
```

When the instance goes to `ERROR`, its fault reads `Build of instance <server-id> aborted: Flavor's disk is too small
for requested image. ...`, and the `nova-compute` log of the compute that tried has:

```text
<base> virtual size <n> larger than flavor root disk size <n>
```

## Impact

Every boot of that image with that flavor, for every user. Running instances are not affected.

## Quick checks

Read-only:

```bash
# 1. The image: file size, virtual size, minimums, format
openstack image show <image> -c size -c virtual_size -c min_disk -c min_ram -c disk_format

# 2. The flavor: root disk (GB) and RAM (MB)
openstack flavor show <flavor> -c disk -c ram
```

Compare: the flavor's `disk` in GB, against the image's `size` and `virtual_size` in bytes and `min_disk` in GB.

## Diagnosis

### Two checks, at two moments (Nova 2026.1)

1. **At the API** (`nova/compute/api.py`, `_validate_flavor_image_nostatus`), before anything is scheduled:
   - the flavor's RAM must be at least the image's `min_ram`;
   - for a local root disk, if the flavor's `disk` is not 0: the image's `size` (the size of the image **file** in
     Glance) and its `min_disk` must both fit in the flavor's disk;
   - for a root disk that is a volume Nova creates: the volume size must be at least `min_disk`;
   - a flavor with `disk` 0 is accepted only for volume-backed instances, unless the policy allows it (admins by
     default): `Only volume-backed servers are allowed for flavors with zero disk.`

   These are refused at once, with HTTP 400 or 403.
2. **On the compute** (`nova/virt/libvirt/imagebackend.py`, `verify_base_size`, for every image backend including
   Ceph RBD), after the image is downloaded: the image's **virtual size** must fit in the flavor's disk. If it does not,
   `nova-compute` logs `<base> virtual size … larger than flavor root disk size …`, and the build is aborted on that
   compute without retrying elsewhere: the instance goes to `ERROR`.

### Why an image passes the first check and fails the second

A compressed image (qcow2) has a file `size` smaller than its `virtual_size`, the size of the disk it holds. The API
compares the file size; the compute compares the virtual size. An image whose `min_disk` is not set, or set too low,
passes the API and fails on the compute. The Nova code says so: `min_disk` "doesn't currently account for virtual disk
size, so we need this extra check here".

## Possible causes

### Cause 1: the flavor is too small for the image

The image's virtual size, or its `min_disk`, is larger than the flavor's root disk.

### Cause 2: the image's `min_disk` is not set or too low

The API lets the boot through, and the compute aborts it after downloading the image.

### Cause 3: the flavor has a zero disk

A flavor with `disk` 0 for an image-backed boot, refused for non-admins.

### Cause 4: the flavor's RAM is below the image's `min_ram`

## Resolution

### For causes 1 and 3: use a flavor that fits, or boot from a volume

Choose a flavor whose `disk` (GB) is at least the image's `virtual_size` (bytes, rounded up to GB) and its `min_disk`.
Or boot from a volume of the right size (`openstack server create --image <image> --boot-from-volume <GB> ...`): the
flavor's disk is then not used for the root disk. With python-openstackclient 9.0.0 that volume is not deleted with
the instance (it stays `available`).

### For cause 2: set `min_disk` on the image

Set `min_disk` to the image's virtual size, rounded up to GB, so the API refuses a too small flavor at once, with a
clear message, instead of an aborted build.

> ⚠️ **State-changing.** Changes the image's metadata: every later boot with a flavor below that size is refused at the
> API. Running instances are not affected. Undo: set the previous value back.

```bash
openstack image set --min-disk <GB> <image>
```

### For cause 4: use a flavor with enough RAM, or correct `min_ram`

`openstack image set --min-ram <MB> <image>` if the image's value is wrong (same warning as above).

## Verification

```bash
openstack server create --image <image> --flavor <flavor> --network <network> --wait <test-name>
openstack server show <test-name> -c status        # ACTIVE
openstack server delete --wait <test-name>
```

> ⚠️ **State-changing.** Creates a test instance, deleted at the end.

## Lab observations

Kolla-Ansible 22.2.0 (2026.1) all-in-one, local qcow2 instance disks, python-openstackclient 9.0.0, on 2026-10-27,
with a CirrOS image resized to a virtual size of 2 GB (file 21 MB) and the 1 GB `m1.tiny` flavor:

- Cause 2 (`min_disk` 0): the API accepted the boot; 12 s later `nova-compute` logged (ERROR,
  `nova.virt.libvirt.imagebackend`) `/var/lib/nova/instances/_base/<base-file> virtual size 2147483648 larger than
  flavor root disk size 1073741824`, then `Build of instance <server-id> aborted: Flavor's disk is too small for
  requested image. Flavor disk is 1073741824 bytes, image is 2147483648 bytes.`; the instance was `ERROR` with that
  message as its fault (code 500) 17 s after the request, with no retry on another host.
- After `openstack image set --min-disk 2`: refused at once, `BadRequestException: 400: Client Error for url:
  <nova>/v2.1/servers, Flavor's disk is smaller than the minimum size specified in image metadata. Flavor disk is
  1073741824 bytes, minimum size is 2147483648 bytes.` With `--boot-from-volume 1`: `Volume is smaller than the
  minimum size specified in image metadata. ...` (400). With `--min-ram 1024`: `Flavor's memory is too small for
  requested image.` (400).
- Cause 3: a flavor with disk 0 was accepted for the admin; a member of a test project got `ForbiddenException:
  403: ... Only volume-backed servers are allowed for flavors with zero disk.`, and booted with
  `--boot-from-volume 1`.
- Resolutions: `--boot-from-volume 3` with `m1.tiny`, and `m1.small`, both `ACTIVE`.
- Not observed: the API refusal for an image file larger than the flavor's disk (`Flavor's disk is too small for
  requested image` at the API), which needs an image file above 1 GB.

## Validity and limits

- Lab-run on Kolla-Ansible 22.2.0 (2026.1) with local disks: causes 2, 3 and 4 and the resolutions (see *Lab
  observations*). Not run: the API refusal on the image file size, and the check on a Ceph RBD image backend.
- Written from Nova 2026.1 (`nova/compute/api.py` `_validate_flavor_image_nostatus`,
  `nova/virt/libvirt/imagebackend.py` `verify_base_size`, `nova/compute/manager.py` (the build abort),
  `nova/exception.py`, the `ZERO_DISK_FLAVOR` policy) and python-openstackclient 2026.1.
- Not covered: resizes to a smaller disk (see [resize not finished](../lifecycle/resize-stuck-in-verify-resize-or-failed.md),
  cause 5), rebuilds, and drivers other than libvirt.

## Related

- [New instance in ERROR with "Exceeded maximum number of retries"](build-fails-exceeded-maximum-number-of-retries.md):
  a flavor or image mismatch aborts the build without retrying.
- [Instance fails to boot from volume](instance-fails-to-boot-from-volume.md)
- Nova 2026.1: [flavors](https://docs.openstack.org/nova/2026.1/user/flavors.html)

---

*Provenance: written by agent:ozarland-author, from the Nova and python-openstackclient 2026.1 source and
documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=scheduling/image-larger-than-flavor-disk.md&title=Feedback%3A+scheduling/image-larger-than-flavor-disk.md):
your release, your deployment method, and what happened.
