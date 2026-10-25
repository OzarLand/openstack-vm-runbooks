---
title: Volume stuck in creating, or in error right after create
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [cinder, glance, rabbitmq]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any multinode
  network: any
  storage: any Cinder backend
  os: any
risk: destructive
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/cinder/2026.1/configuration/block-storage/scheduler-filters.html
  - https://docs.openstack.org/cinder/2026.1/admin/image-volume-cache.html
  - https://docs.openstack.org/api-ref/block-storage/v3/#messages-messages
  - https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/volume.html
authored_by: agent:ozarland-author
last_verified: 2026-10-25
---

# Volume stuck in creating, or in error right after create

## Symptom

A user creates a volume (`openstack volume create --size <n> …`, possibly `--image`, `--snapshot` or `--type`). The
request is accepted, but the volume never becomes `available`:

```text
$ openstack volume show <volume-id> -c status -c os-vol-host-attr:host -c volume_type -c created_at
| status                | error     |              # within seconds of the request
...
| status                | creating  |              # or downloading, minutes or hours after the request
```

A request refused at once (quota exceeded, invalid volume type, size too small for the image) returns an error and
creates no volume: that is not this page (`openstack quota show <project-id>` shows the volume and gigabytes limits).

## Impact

The volumes being created, for one project or for everyone, depending on the cause. Existing volumes and their
attachments are not affected. A volume in `error` or stuck in `creating` holds no user data yet, but it counts
against the project's quota until it is deleted. The risk comes from the fixes: forcing a status by hand can hide a
volume that the backend is in fact still creating.

## Quick checks

Read-only. With admin credentials, from any host with the OpenStack client:

```bash
# 1. Status, the backend chosen (empty if no backend was chosen) and the type
openstack volume show <volume-id> -c status -c os-vol-host-attr:host -c volume_type -c size -c created_at

# 2. Cinder's own explanation (volume API 3.3 or later), for this volume
openstack --os-volume-api-version 3.3 volume message list | grep <volume-id>

# 3. Are the Cinder services up? A backend's cinder-volume is listed as <host>@<backend>.
openstack volume service list

# 4. What each pool has left, as the scheduler sees it
openstack volume backend pool list --long

# 5. The volume type's properties: volume_backend_name must match a configured backend
openstack volume type show <type> -c properties
```

How to read them:

| You see | Go to |
|---|---|
| `error`, empty host, check 2 says `Could not find any available weighted backend.` | [Cause 1](#cause-1-no-backend-passes-the-scheduler) |
| `error`, a host is set, check 2 says `Driver failed to create the volume.` or `Image used for creating volume exceeds available space.` | [Cause 2](#cause-2-the-backend-failed-to-create-it) |
| `creating` or `downloading` for longer than the image size explains, and the `cinder-volume` of its host is `down` in check 3 | [Cause 3](#cause-3-the-service-that-had-the-request-stopped) |
| `creating`, empty host, every service `up` | [Cause 4](#cause-4-the-request-never-reached-a-service) |
| `downloading`, the service `up`, the log still copying | [Cause 5](#cause-5-the-image-is-still-being-written) |

## Diagnosis

### Cinder's path for a new volume

`cinder-api` creates the volume record in `creating` and casts the request to `cinder-scheduler` over RabbitMQ. The
scheduler filters the backends (by default `AvailabilityZoneFilter`, `CapacityFilter` and `CapabilitiesFilter`),
weighs the rest, records the chosen backend as `os-vol-host-attr:host` and casts to that backend's `cinder-volume`.
`cinder-volume` asks the driver to create the volume. From an image, it sets the status to `downloading`, then
writes the image to the volume. On success, the status becomes `available`. If the scheduler finds no backend, it
sets `error` itself, which is why `error` within seconds and an empty host point to the scheduler.

A `cinder-volume` or `cinder-scheduler` service that starts again sets the volumes it had left in `creating` or
`downloading` to `error`: it does not resume them.

### Logs

Search for the volume ID:

**Kolla-Ansible:** `/var/log/kolla/cinder/cinder-scheduler.log` on the hosts running `cinder_scheduler`, and
`/var/log/kolla/cinder/cinder-volume.log` on the hosts running `cinder_volume` (the `cinder` group, the control
hosts by default in 2026.1).

**Any:** the `cinder-scheduler` and `cinder-volume` service logs.

What to look for:

- in the scheduler log: `No valid backend was found`, with the filter that rejected the backends (for example
  `CapacityFilter` or `CapabilitiesFilter`);
- in the volume log: the driver's exception (LVM, RBD, NFS, a vendor driver), or the image download and conversion;
- no trace of the volume ID in either log: the request was lost before reaching a service (cause 4).

## Possible causes

### Cause 1: no backend passes the scheduler

- **Capacity:** no pool has enough free space for the size, once `reserved_percentage` and, for thin provisioning,
  `max_over_subscription_ratio` are applied. Check 4 shows `free_capacity_gb` per pool.
- **Capabilities:** the volume type's `volume_backend_name` (or another extra spec) matches no configured backend,
  for example after a backend was renamed in `cinder.conf`.
- **Availability zone:** the requested zone has no `cinder-volume` service.
- **Backend down:** the only matching backend's `cinder-volume` is `down`, so the scheduler does not see it.

### Cause 2: the backend failed to create it

The backend was chosen and the driver failed: an LVM volume group full (`vgs` on the `cinder-volume` host shows its
free space), a Ceph pool full or unreachable, an NFS share not mounted, a vendor array refusing the request. For a
volume from an image: no space in `image_conversion_dir` on the `cinder-volume` host to download or convert the
image, or a Glance error.

### Cause 3: the service that had the request stopped

`cinder-volume` stopped or lost RabbitMQ while it was creating the volume. The volume stays `creating` or
`downloading` until that service starts again, which then sets it to `error`.

### Cause 4: the request never reached a service

The record exists, but the cast to the scheduler or to `cinder-volume` was lost (RabbitMQ unavailable or
partitioned at that moment). Nothing will move the volume: no service knows about it.

### Cause 5: the image is still being written

Not a failure. A large image, or one that must be converted to raw, takes time to download and write. The
`cinder-volume` log shows the copy progressing. Cinder's image-volume cache makes later volumes from the same image
much faster.

## Resolution

Work by risk. The quick checks are read-only. Then fix the cause, from the least to the most intrusive step. Most
of the time a volume in `error` cannot be retried: fix the cause and create a new volume.

### For cause 1

Read-only first: compare the requested size with `free_capacity_gb` (check 4), and the type's `volume_backend_name`
with the `volume_backend_name` of each backend in `cinder.conf`. Then:

- capacity: free space on the backend, or add one; raising `max_over_subscription_ratio` on a thin backend is a
  capacity decision, not a fix;
- a type that names no backend: correct the type's property (a change for every future volume of that type);

> ⚠️ **State-changing.** Changes the volume type for every future volume of that type; existing volumes keep their
> backend. Undo: set the previous value back.

```bash
openstack volume type set --property volume_backend_name=<backend-name> <type>
```

- a backend `down`: see cause 3.

### For cause 2

Fix the backend (space in the volume group or pool, the NFS mount, the array) or the conversion directory, as the
`cinder-volume` log says. Then create a new volume.

### For cause 3

Find why the service stopped (its log, RabbitMQ) and start it again. On start, it sets the volumes it had left in
`creating` or `downloading` to `error`.

> ⚠️ **State-changing.** Restarts the `cinder-volume` service of this host. Volumes being created or attached
> through it at that moment can fail and need retrying; volumes already attached to instances stay attached. Fix
> what stopped it first: a restart does not fix a full backend or a broken RabbitMQ.

**Kolla-Ansible:**

```bash
sudo systemctl restart kolla-cinder_volume-container.service
```

**Any:**

```bash
sudo systemctl restart <cinder-volume-unit>
```

### For cause 4

Check RabbitMQ first (`docker exec rabbitmq rabbitmqctl cluster_status` on Kolla-Ansible). If the volume has no host
and no service knows it, nothing will finish it: its status has to be set by hand, then the volume deleted.

> 🛑 **Destructive.** Forces Cinder's record of the volume to `error`. If a backend is in fact still creating it
> (a host is set and that service is working on it), the record then lies about the backend, and the delete below can
> fail or leave an orphan on the storage. Precondition: `os-vol-host-attr:host` is empty, or the `cinder-volume` log
> of that host shows no activity for the volume ID; and the volume was never `available`.

```bash
openstack volume set --state error <volume-id>
```

### Clean up volumes left in error

A volume in `error` holds no user data when it was never `available`. Delete it to give the quota back:

> 🛑 **Destructive.** Deletes the volume and, if the backend created anything, its storage. Precondition: the
> volume was never `available` (check its `created_at` and the logs) and is not attached (`attachments` empty).
> Cannot be undone.

```bash
openstack volume delete <volume-id>
```

### For cause 5

Wait, and follow the copy in the `cinder-volume` log. For the next volumes from that image, enable the
[image-volume cache](https://docs.openstack.org/cinder/2026.1/admin/image-volume-cache.html) on the backend, and,
with Ceph RBD for both Glance and Cinder, keep images in `raw` so that Cinder clones them instead of copying.

## Verification

```bash
openstack volume service list
# expected: every cinder-scheduler and cinder-volume service up

openstack volume create --size 1 runbook-check
openstack volume show runbook-check -c status -c os-vol-host-attr:host
# expected: available, on the expected backend
openstack volume delete runbook-check
```

The verification volume is created and deleted on purpose:

> ⚠️ **State-changing.** Creates a 1 GB test volume, deleted right after. Needs volume and gigabytes quota.

## Validity and limits

- Written from Cinder 2026.1 (the scheduler and volume managers, their clean-up on start, the default scheduler
  filters, user messages, `service_down_time`) and Kolla-Ansible 2026.1. Not reproduced on a lab yet.
- The user messages quoted come from `cinder/message/message_field.py`; how they appear depends on the client.
- Not covered: volumes created by Nova for a boot from volume (see the related runbook), backups and restores,
  snapshots stuck in `creating`, replication, multi-attach, encrypted volumes, and migrations or retypes stuck in
  `migrating` or `retyping`.

## Related

- [Instance fails to boot from volume](https://github.com/OzarLand/openstack-vm-runbooks/blob/main/scheduling/instance-fails-to-boot-from-volume.md): the same failures seen
  from Nova, when it creates the volume.
- [Volume stuck in attaching, reserved or detaching](volume-stuck-in-attaching-or-detaching.md)
- [Stale volume attachment after the instance was deleted or evacuated](stale-volume-attachment-after-instance-deleted-or-evacuated.md)
- Cinder 2026.1: [scheduler filters](https://docs.openstack.org/cinder/2026.1/configuration/block-storage/scheduler-filters.html),
  [image-volume cache](https://docs.openstack.org/cinder/2026.1/admin/image-volume-cache.html),
  [messages API](https://docs.openstack.org/api-ref/block-storage/v3/#messages-messages)

---

*Provenance: written by agent:ozarland-author, from the Cinder and Kolla-Ansible 2026.1 documentation and source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=volumes/volume-stuck-in-creating-or-error-after-create.md&title=Feedback%3A+volumes/volume-stuck-in-creating-or-error-after-create.md):
your release, your deployment method, and what happened.
