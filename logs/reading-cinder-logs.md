---
title: Reading the Cinder logs. cinder-api, cinder-scheduler, cinder-volume, and os-brick on the compute
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [cinder, nova]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-27, note: "Cinder LVM over iSCSI (tgtd), no multipath: normal create, start-up, cinder-volume stopped (scheduler lines), queued delete after restart, LVM lvremove in use, attach and detach (os-brick) captured; not observed: driver init failure, heartbeat stop, create failed, image copy failure, terminate connection failure, DB loss, capacity filter lines, os-brick failure lines" }
upstream_refs:
  - https://opendev.org/openstack/cinder/src/branch/stable/2026.1/cinder/volume/manager.py
  - https://opendev.org/openstack/cinder/src/branch/stable/2026.1/cinder/volume/flows/manager/create_volume.py
  - https://opendev.org/openstack/cinder/src/branch/stable/2026.1/cinder/scheduler/filter_scheduler.py
  - https://opendev.org/openstack/cinder/src/branch/stable/2026.1/cinder/scheduler/host_manager.py
  - https://opendev.org/openstack/cinder/src/branch/stable/2026.1/cinder/scheduler/base_filter.py
  - https://opendev.org/openstack/cinder/src/branch/stable/2026.1/cinder/scheduler/filters/capacity_filter.py
  - https://opendev.org/openstack/cinder/src/branch/stable/2026.1/cinder/service.py
  - https://opendev.org/openstack/os-brick/src/branch/stable/2026.1/os_brick/initiator/connectors/iscsi.py
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Reading the Cinder logs. cinder-api, cinder-scheduler, cinder-volume, and os-brick on the compute

A volume operation crosses four logs: `cinder-api` accepts it, `cinder-scheduler` picks a backend, `cinder-volume`
talks to the storage, and for an attach, `nova-compute` connects the host to the volume through `os-brick`. This page
says what each logs and what the common lines mean. Messages are quoted from Cinder and os-brick 2026.1. Ceph client
and iSCSI/multipath system logs are covered on [reading the storage logs of a compute](host-storage-logs-iscsi-multipath-nfs-rbd.md).

Runbooks for the fixes: [stuck in creating or error](../volumes/volume-stuck-in-creating-or-error-after-create.md),
[stuck in attaching or detaching](../volumes/volume-stuck-in-attaching-or-detaching.md).

## Where the logs are

**Kolla-Ansible:** `/var/log/kolla/cinder/cinder-api.log` (requests also in `cinder-api-uwsgi.log`),
`cinder-scheduler.log` and `cinder-volume.log` (`log_dir = /var/log/kolla/cinder` in `cinder.conf.j2`), on the nodes
that run each service. The `os-brick` lines of an attach are in `/var/log/kolla/nova/nova-compute.log` on the compute.

**Any:** `/var/log/cinder/` or the journal; `os-brick` logs in the log of the process that uses it (`nova-compute`).

Cinder passes the volume to many log calls as a `resource`, but the default `oslo.log` format does not print
`%(resource)s` (it is not in `logging_context_format_string`), so lines such as `Created volume successfully.` do not
show the volume id. Follow a volume by **request id** (the `x-openstack-request-id` of the call, or Nova's request
id, which Cinder logs as the global request id for calls Nova makes;
[where the logs are](where-openstack-logs-are-and-how-to-read-a-line.md#request-id-and-global-request-id)), and by
volume id for the lines that put it in their message text (scheduler, `Volume <id>: create failed`, Nova's lines).

```bash
# Kolla-Ansible, on the controllers and storage nodes (the directory is readable by root only, so the glob must be
# expanded by a root shell)
sudo sh -c 'grep -h "<volume-id>" /var/log/kolla/cinder/cinder-*.log' | sort
openstack volume service list          # cinder-scheduler and cinder-volume (<host>@<backend>) up or down
```

## cinder-volume

### What normal looks like

At start, for each backend (`volume/manager.py`):

| Line (message text) | Level |
|---|---|
| `Starting volume driver <driver class> (<version>)` | INFO |
| `Driver initialization completed successfully.` | INFO |
| `Initializing RPC dependent components of volume driver <driver class> (<version>)` | INFO |

Then for each operation, `Created volume successfully.` or `Deleted volume successfully.` (INFO; the volume id is
not in the line with the default format, the request id is).

### Lines that signal a problem

| Line (message text) | Level | What it means | Next |
|---|---|---|---|
| `Failed to initialize driver.` with a traceback | ERROR | The backend driver could not start: the storage is unreachable, credentials are wrong, or a pool is missing. The backend does not serve requests. | the traceback; the storage system |
| `Manager for service cinder-volume <host>@<backend> is reporting problems, not sending heartbeat. Service will appear "down".` | ERROR | The manager knows its driver is not working, so it stops its heartbeat on purpose: `volume service list` shows it `down`, and the scheduler skips it. | the driver initialization lines above |
| `Volume <volume-id>: create failed` | ERROR | The create flow failed; the volume goes to `error`. The traceback before it gives the cause (driver error, image copy failure). | the traceback |
| `Failed to copy image to volume: <reason>` | (exception text) | Creating a volume from an image failed during the image download or conversion. | Glance's log for the image; disk space for the conversion |
| `Image download <n> MB at <n> MB/s` | INFO | Speed of an image download into a volume. | — |
| `Terminate volume connection failed: <error>` | ERROR (with traceback) | Removing the host's access to the volume at detach time failed on the storage side. | [stuck in detaching](../volumes/volume-stuck-in-attaching-or-detaching.md) |
| `model server went away` with a traceback, then `Recovered model server connection!` | ERROR | The service could not write its heartbeat to the database, then recovered. | MariaDB/Galera |

Sources: `cinder/volume/manager.py`, `cinder/volume/flows/manager/create_volume.py`, `cinder/service.py`,
`cinder/exception.py`, `cinder/image/image_utils.py`.

## cinder-scheduler

| Line (message text) | Level | What it means | Next |
|---|---|---|---|
| `volume service is down. (host: <host>@<backend>)` | WARNING | A backend's heartbeat is older than `service_down_time` (default 60 s; `report_interval` is 10 s). The scheduler ignores it. | that backend's `cinder-volume.log` |
| `Filtering removed all hosts for the request with volume ID '<volume-id>'. Filter results: <filter>: (start: <n>, end: <n>), ...` | INFO | Backends were up but the filters removed them all; the filter whose `end` is 0 is the cause (capacity, volume type extra specs, availability zone). | the backend's reported capabilities |
| `Insufficient free virtual space (<n>GB) to accommodate thin provisioned <n>GB volume on host <host>@<backend>#<pool>.` (`cluster <name>` when the backend is clustered) | WARNING | The capacity filter rejected a thin-provisioned backend: its total capacity less `reserved_percentage`, times `max_over_subscription_ratio`, minus what is already provisioned, is below the size asked. The usual cause of `CapacityFilter: (start: <n>, end: 0)`. Related lines from the same filter: `Insufficient free space for thin provisioning. …` (over-subscription ratio exceeded) and `Insufficient free space for volume creation on host … (requested / avail): <n>/<n>`. | `openstack volume backend pool list --long`; `max_over_subscription_ratio` |
| `No weighed backend found for volume with properties: <volume type>` | WARNING | No backend left after filtering and weighing. | same |
| `Failed to schedule_<method>: <error>` (extend, retype, migrate, manage; for example `Failed to schedule_extend_volume: Cannot place volume <volume-id> on <host>@<backend>#<pool>.`) | ERROR | Scheduling of that operation failed. | the lines just above |
| `Failed to run task cinder.scheduler.flows.create_volume.ScheduleCreateVolumeTask;volume:create: <error>` | ERROR | The create's scheduling failed; the volume goes to `error` with no host. | the lines just above |

Sources: `cinder/scheduler/host_manager.py`, `cinder/scheduler/base_filter.py`,
`cinder/scheduler/filter_scheduler.py`, `cinder/scheduler/manager.py`, `cinder/scheduler/filters/capacity_filter.py`,
`cinder/exception.py` (`NoValidBackend`).

## On the compute: os-brick during attach and detach

`nova-compute` uses `os-brick` to connect the host to the volume (iSCSI, FC, RBD, NVMe-oF…). Its lines are in
`nova-compute.log`, with the `os_brick` module name. For iSCSI (`os_brick/initiator/connectors/iscsi.py`):

| Line (message text) | Level | What it means |
|---|---|---|
| `Multipath discovery for iSCSI enabled` / `not enabled.` | INFO | Whether the attach uses multipath (`[libvirt] volume_use_multipath` in Nova). |
| `Failed to connect to iSCSI portal <portal>.` | WARNING | Login to one target portal failed (network, CHAP, target down). With multipath, other portals may still work. |
| `LUN <lun> on iSCSI portal <portal> not found on sysfs after logging in.` | WARNING | Logged in, but the expected LUN did not appear: the storage did not map the volume to this host (initiator name, host group). |
| `No dm was created, connection to volume is probably bad and will perform poorly.` | WARNING | Multipath requested but no multipath device was built: all paths failed or `multipathd` is not running. |
| `There were errors removing <devices>, leftovers may remain in the system` | WARNING | Detach could not cleanly remove the SCSI devices; stale devices remain on the host. |
| `Volume device not found at <device>.` | (exception text) | The device never appeared; the attach fails. |

Sources: `os_brick/initiator/connectors/iscsi.py`, `os_brick/exception.py`.

```bash
# On the compute
sudo grep -n "os_brick" /var/log/kolla/nova/nova-compute.log | grep -E "WARNING|ERROR" | tail -20
sudo grep -n "<volume-id>" /var/log/kolla/nova/nova-compute.log      # Nova's attach/detach lines for this volume
```

Nova logs `Attaching volume <volume-id> to <mountpoint>` and `Detaching volume <volume-id>` (INFO,
`nova/compute/manager.py`) around these lines.

## Captured on a lab

Kolla-Ansible 22.2.0 (2026.1, Cinder 28.0.1) all-in-one, LVM (thin) over iSCSI with tgtd, backend `lvm-1`, Nova
`volume_use_multipath` off, 2026-10-27. Request, user and project ids replaced.

A normal create, `cinder-volume.log`. The two `create_volume` lines carry the volume id; `Created volume
successfully.` does not:

```text
INFO cinder.volume.flows.manager.create_volume [None req-<request-id> <id> <id> - - - -] Volume <volume-id>: being created as raw with specification: {'status': 'creating', 'volume_name': 'volume-<volume-id>', 'volume_size': 1}
INFO cinder.volume.flows.manager.create_volume [None req-<request-id> <id> <id> - - - -] Volume volume-<volume-id> (<volume-id>): created successfully
INFO cinder.volume.manager [None req-<request-id> <id> <id> - - - -] Created volume successfully.
```

Start-up after `systemctl restart kolla-cinder_volume-container.service` (19 s to the last line):

```text
INFO cinder.service [-] Starting cinder-volume node (version 28.0.1)
INFO cinder.volume.manager [None req-<request-id> - - - - - -] Starting volume driver LVMVolumeDriver (3.0.0)
INFO cinder.volume.manager [None req-<request-id> - - - - - -] Driver initialization completed successfully.
INFO cinder.volume.manager [None req-<request-id> - - - - - -] Initializing RPC dependent components of volume driver LVMVolumeDriver (3.0.0)
INFO cinder.volume.manager [None req-<request-id> - - - - - -] Driver post RPC initialization completed successfully.
```

`cinder-volume` stopped, then a create (`cinder-scheduler.log`; the volume goes to `error` with no host). For a
create the last line is `Failed to run task ...`, not `Failed to schedule_create_volume`; `Failed to schedule_<method>`
was seen for an extend (`Failed to schedule_extend_volume: Cannot place volume <volume-id> on <host>@lvm-1#lvm-1.`):

```text
WARNING cinder.scheduler.host_manager [...] volume service is down. (host: aio-1@lvm-1)
INFO cinder.scheduler.base_filter [...] Filtering removed all hosts for the request with volume ID '<volume-id>'. Filter results: AvailabilityZoneFilter: (start: 0, end: 0), ...
WARNING cinder.scheduler.filter_scheduler [...] No weighed backend found for volume with properties: {'id': '<type-id>', 'name': '__DEFAULT__', ...
ERROR cinder.scheduler.flows.create_volume [...] Failed to run task cinder.scheduler.flows.create_volume.ScheduleCreateVolumeTask;volume:create: No valid backend was found. No weighed backends ...
```

Both blocks below were captured during the lab run of
[#133](https://github.com/OzarLand/openstack-vm-runbooks/pull/133) (see the *Lab observations* of
[Volume stuck in deleting or error_deleting](../volumes/volume-stuck-in-deleting-or-error-deleting.md)).

After the restart of a stopped `cinder-volume`, with a delete queued:

```text
INFO cinder.manager [None req-<request-id> - - - - - -] Initiating service 2 cleanup
INFO cinder.manager [None req-<request-id> - - - - - -] Service 2 cleanup completed.
INFO cinder.volume.drivers.lvm [None req-<request-id> <id> <id> - - - -] Successfully deleted volume: <volume-id>
INFO cinder.volume.manager [None req-<request-id> <id> <id> - - - -] Deleted volume successfully.
```

A delete that the LVM driver could not run (the logical volume open on the host; the volume goes to `error_deleting`):

```text
ERROR oslo_messaging.rpc.server [None req-<request-id> <id> <id> - - - -] Exception during message handling: oslo_concurrency.processutils.ProcessExecutionError: Unexpected error while running command.
ERROR oslo_messaging.rpc.server Command: sudo cinder-rootwrap /etc/cinder/rootwrap.conf lvremove --config activation { retry_deactivation = 1} devices { ignore_suspended_devices = 1} -f cinder-volumes/volume-<volume-id>
ERROR oslo_messaging.rpc.server Stderr: '  Logical volume cinder-volumes/volume-<volume-id> in use.\n'
```

An attach, `nova-compute.log` (the detach logged no `os_brick` line at INFO). The `nvmeof` WARNING and the `lightos`
INFO line come from os-brick probing every connector and are harmless on an iSCSI-only host; `No active sessions.` is
logged before the first login of the host:

```text
INFO nova.compute.manager [...] [instance: <server-id>] Attaching volume <volume-id> to /dev/vdb
WARNING os_brick.initiator.connectors.nvmeof [...] Could not find nvme_core/parameters/multipath: FileNotFoundError: [Errno 2] No such file or directory: '/sys/module/nvme_core/parameters/multipath'
INFO os_brick.initiator.connectors.lightos [...] Current host hostNQN nqn.2014-08.org.nvmexpress:uuid:<uuid> and IP(s) are [...]
INFO os_brick.initiator.connectors.iscsi [...] Trying to connect to iSCSI portal 192.0.2.10:3260
WARNING os_brick.initiator.connectors.iscsi [...] iscsiadm stderr output when getting sessions: iscsiadm: No active sessions.
INFO nova.compute.manager [...] [instance: <server-id>] Detaching volume <volume-id>
```

## Reading a volume problem in order

1. **Volume status and service list**: `error` at create → scheduler then volume logs; `attaching`/`detaching` →
   nova-compute (os-brick) and the volume's backend.
2. **cinder-scheduler** for the volume id: did it pick a backend?
3. **cinder-volume** on that backend: driver errors, create failed.
4. **nova-compute** on the instance's host for an attach: os-brick warnings, then libvirt.

## Validity and limits

- The tables quote Cinder and os-brick 2026.1. The lines under *Captured on a lab* are real (all-in-one, LVM over
  iSCSI, 2026-10-27); not observed on the lab: `Failed to initialize driver.`, `is reporting problems, not sending
  heartbeat`, `create failed`, `Failed to copy image to volume`, `Terminate volume connection failed`, `model server
  went away`, the capacity filter's `Insufficient ...` lines and the os-brick failure lines.
- Driver-specific messages (Ceph RBD, LVM, vendor drivers) are not listed; each driver logs its own.
- Only the iSCSI connector of os-brick is detailed.

## Related

- [Volume stuck in creating or error after create](../volumes/volume-stuck-in-creating-or-error-after-create.md)
- [Volume stuck in attaching or detaching](../volumes/volume-stuck-in-attaching-or-detaching.md)
- [Instance fails to boot from volume](../scheduling/instance-fails-to-boot-from-volume.md)
- [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md)
- [Logs index](README.md)

---

*Provenance: written by agent:ozarland-author, from Cinder and os-brick 2026.1 source code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Freading-cinder-logs.md&title=Feedback%3A+logs%2Freading-cinder-logs.md):
your release, your deployment method, and what happened.
