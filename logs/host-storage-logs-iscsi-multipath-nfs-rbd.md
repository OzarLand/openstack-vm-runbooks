---
title: Reading the storage logs of a compute. iscsid, multipathd, NFS client messages and Ceph client (librbd) logs
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [nova, cinder, iscsi, multipath, nfs, ceph]
deployment_methods: [kolla-ansible]
environment:
  topology: any
  network: any
  storage: iscsi, nfs or ceph-rbd volumes attached to instances
  os: any linux with systemd
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-27, note: "iSCSI without multipath only: port 3260 dropped for 60 s and 180 s (replacement_timeout 120 s, noop 5/5 s), kernel and iscsid lines captured, read commands run; not run: multipath (not deployed), NFS (no backend), Ceph RBD client logs and their state-changing setup (no Ceph)" }
upstream_refs:
  - https://github.com/open-iscsi/open-iscsi/blob/2.1.13/README
  - https://github.com/open-iscsi/open-iscsi/blob/2.1.13/usr/initiator.c
  - https://github.com/open-iscsi/open-iscsi/blob/2.1.13/usr/iscsid.c
  - https://github.com/open-iscsi/open-iscsi/blob/2.1.13/usr/log.c
  - https://github.com/open-iscsi/open-iscsi/blob/2.1.13/include/iscsi_if.h
  - https://github.com/torvalds/linux/blob/v7.3/drivers/scsi/scsi_transport_iscsi.c
  - https://github.com/torvalds/linux/blob/v7.3/include/scsi/scsi_transport_iscsi.h
  - https://github.com/torvalds/linux/blob/v7.3/net/sunrpc/clnt.c
  - https://github.com/opensvc/multipath-tools/blob/0.15.1/multipathd/multipathd.8.in
  - https://github.com/opensvc/multipath-tools/blob/0.15.1/multipathd/main.c
  - https://github.com/opensvc/multipath-tools/blob/0.15.1/libmultipath/structs_vec.c
  - https://github.com/ceph/ceph/blob/tentacle/doc/rbd/rbd-openstack.rst
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/iscsi/templates/iscsid.json.j2
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/multipathd/templates/multipathd.json.j2
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/group_vars/all/iscsi.yml
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/group_vars/all/multipathd.yml
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/nova-cell/defaults/main.yml
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/conf/libvirt.py
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Reading the storage logs of a compute. iscsid, multipathd, NFS client messages and Ceph client (librbd) logs

When a Cinder volume is attached to an instance, the compute itself is the storage client: an iSCSI session (often
under multipath), an NFS mount, or librbd inside the QEMU process. When the storage path breaks, the instance freezes
or sees I/O errors, and Nova and Cinder often log nothing. The explanation is in the logs of the compute's storage
client. This page shows where they are and which lines matter.

The kernel's block layer and filesystem errors, and hung tasks, are on the page
[reading the kernel log of a compute](kernel-messages-on-computes.md). The errors that `os-brick` raises in
`nova-compute.log` during an attach are covered on [reading the Cinder logs](reading-cinder-logs.md) and with the
volume error messages; for a stuck attach, see
[volume stuck in attaching or detaching](../volumes/volume-stuck-in-attaching-or-detaching.md).

All commands on this page are read-only, except the Ceph client logging setup, which carries its own warning.

## iSCSI: iscsid and the kernel

### Where the logs are

**Kolla-Ansible:** the `iscsid` container is deployed when Cinder uses an iSCSI backend (`enable_iscsid` in
[`group_vars/all/iscsi.yml`](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/group_vars/all/iscsi.yml)).
It runs `iscsid -d 8 -f`
([`iscsid.json.j2`](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/iscsi/templates/iscsid.json.j2)):
in the foreground, at the highest debug level. In the foreground, `iscsid` writes its messages to standard output
and standard error instead of syslog (`log_do_log_std` in
[`usr/log.c`](https://github.com/open-iscsi/open-iscsi/blob/2.1.13/usr/log.c)), so they end up in the container's
output:

```bash
# On the compute
journalctl -u kolla-iscsid-container.service --since "-1h"
docker logs --since 1h iscsid 2>&1 | tail -200     # Podman: podman logs
```

Expect a lot of debug output; filter for the lines below.

**Any (package install):** `iscsid` runs as a daemon and logs to syslog: `journalctl -u iscsid`.

The kernel side of iSCSI logs to the kernel log (`journalctl -k`).

### The lines that matter

Format strings, not captured lines:

| Line (format) | Where | Meaning |
|---|---|---|
| `connection<sid>:<cid>: detected conn error (<code>)` | Kernel ([`scsi_transport_iscsi.c`](https://github.com/torvalds/linux/blob/v7.3/drivers/scsi/scsi_transport_iscsi.c)) | The connection failed. From this moment, the `replacement_timeout` runs |
| `Kernel reported iSCSI connection <sid>:<cid> error (<code> - <name>) state (<n>)` | iscsid ([`usr/initiator.c`](https://github.com/open-iscsi/open-iscsi/blob/2.1.13/usr/initiator.c)) | iscsid's view of the same error, with the name of the code |
| `connection<sid>:<cid> is operational after recovery (<n> attempts)` | iscsid | The session came back |
| `session<sid>: session recovery timed out after <n> secs` | Kernel | `replacement_timeout` expired: the I/O waiting on this session is failed up to multipath, the filesystem or the instance |
| `Login failed ...` (several variants) | iscsid ([`usr/login.c`](https://github.com/open-iscsi/open-iscsi/blob/2.1.13/usr/login.c)) | The target refused or could not complete the login (authentication, target not found) |

The `<code>` is `1000` (`ISCSI_ERR_BASE`) plus the error number of
[`include/iscsi_if.h`](https://github.com/open-iscsi/open-iscsi/blob/2.1.13/include/iscsi_if.h). The ones seen most
often:

| Code | Name | Meaning (from open-iscsi's README, section 8.1.3) |
|---|---|---|
| 1011 | `ISCSI_ERR_CONN_FAILED` | Generic: a PDU timed out, the network returned an error, or a link went down |
| 1020 | `ISCSI_ERR_TCP_CONN_CLOSE` | The target closed the connection |
| 1021 | `ISCSI_ERR_SCSI_EH_SESSION_RST` | A SCSI command timed out, the SCSI error handler tried to abort it (and possibly a LUN reset), and now drops the session |
| 1022 | `ISCSI_ERR_NOP_TIMEDOUT` | An iSCSI ping (NOP-Out) got no answer in time |

The [open-iscsi README](https://github.com/open-iscsi/open-iscsi/blob/2.1.13/README) (section 8.1) explains the
timers involved: NOP-Out every `noop_out_interval` (10 s by default), failing after `noop_out_timeout` (15 s), then
`replacement_timeout` (120 s by default) before I/O is failed. With dm-multipath it advises shorter values, so that I/O
moves quickly to another path. Many `NOP_TIMEDOUT` errors can mean timers too aggressive for the network, or a network
problem.

```bash
# On the compute
journalctl -k | grep -E "detected conn error|session recovery timed out"   # current boot only: add -b -1 after a reboot
iscsiadm -m session -P 1          # sessions, their state and target portal (Kolla-Ansible: docker exec iscsid ...)
```

`journalctl -k` reads only the current boot, even with `--since`: after the compute rebooted, the lines of the
previous boot need `journalctl -k -b -1` (`journalctl --list-boots` lists the boots).

Captured on a lab (Kolla-Ansible 2026.1 all-in-one, kernel 6.8, open-iscsi 2.1.9, `noop_out_interval` and
`noop_out_timeout` 5 s, `replacement_timeout` 120 s, port 3260 dropped on the compute for 60 s, then for 180 s):

```text
compute-1 kernel:  connection5:0: ping timeout of 5 secs expired, recv timeout 5, last rx <n>, last ping <n>, now <n>
compute-1 kernel:  connection5:0: detected conn error (1022)
compute-1 docker[<pid>]: iscsid: Kernel reported iSCSI connection 5:0 error (1022 - ISCSI_ERR_NOP_TIMEDOUT: A NOP has timed out) state (3)
compute-1 docker[<pid>]: iscsid: re-opening session 5 (reopen_cnt 0)
compute-1 docker[<pid>]: iscsid: connection5:0 login failed ISCSI_CONN_STATE_XPT_WAIT/R_STAGE_SESSION_REOPEN (reopen_cnt=0, reopen_max=0)
compute-1 docker[<pid>]: iscsid: connection5:0 is operational after recovery (4 attempts)
compute-1 kernel:  session5: session recovery timed out after 120 secs
compute-1 kernel: sd 2:0:0:1: rejecting I/O to offline device
compute-1 kernel: I/O error, dev sda, sector 1049536 op 0x1:(WRITE) flags 0x8800 phys_seg 2 prio class 2
```

On that lab: the error was detected 8 s after the cut; during it `iscsiadm -m session -P 1` showed `iSCSI Connection
State: TRANSPORT WAIT`, `iSCSI Session State: FAILED`; the guest's writes blocked without error for a 60 s cut and
resumed 1 s after recovery; for the 180 s cut, `session recovery timed out` came 120 s after detection, the guest got
I/O errors and remounted its ext4 file system read-only, and stayed so after the session came back. `nova-compute`
and `cinder-volume` logged nothing. The iscsid output had 1387 lines in 100 s (`-d 8`); `Login failed` with a capital
L never appeared, only the lower-case `login failed ...` above, one every 14 to 15 s.

## Multipath: multipathd

### Where the logs are

**Kolla-Ansible:** the `multipathd` container is off by default (`enable_multipathd: false` in
[`group_vars/all/multipathd.yml`](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/group_vars/all/multipathd.yml)).
When enabled, it runs `/sbin/multipathd -d`
([`multipathd.json.j2`](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/multipathd/templates/multipathd.json.j2)),
which does not daemonise and prints all messages to stdout and stderr
([`multipathd(8)`](https://github.com/opensvc/multipath-tools/blob/0.15.1/multipathd/multipathd.8.in)):

```bash
journalctl -u kolla-multipathd-container.service --since "-1h"
```

**Any:** `journalctl -u multipathd`.

### The lines that matter

Format strings from multipath-tools 0.15.1 (`<map>` is the multipath device alias or WWID, `<path>` a device such as
`sdb` or its `major:minor`):

| Line (format) | Meaning | Source |
|---|---|---|
| `checker failed path <path> in map <map>` | The path checker found this path down | `multipathd/main.c` |
| `<path>: mark as failed` | The path is failed in the map | `multipathd/main.c` |
| `<map>: remaining active paths: <n>` | How many paths are left. `0` means no path to the volume | `libmultipath/structs_vec.c` |
| `<path>: reinstated` | The path is back | `multipathd/main.c` |
| `<map>: Entering recovery mode: max_retries=<n>` | No path left, `no_path_retry` is a number: I/O is queued for that many checker intervals | `libmultipath/structs_vec.c` |
| `<map>: Disable queueing` | The retries ran out: queued I/O now fails | `multipathd/main.c` |
| `<map>: queue_if_no_path enabled`, `<map>: Recovered to normal mode` | A path came back during recovery | `libmultipath/structs_vec.c` |

While multipath queues I/O with no path, the instance's disk is frozen, not failed; after `Disable queueing`, the
guest sees I/O errors.

```bash
journalctl -u multipathd -u kolla-multipathd-container.service | grep -E "checker failed|remaining active paths|reinstated|recovery mode|Disable queueing"
sudo multipath -ll                # maps, paths and their states (Kolla-Ansible: docker exec multipathd ...)
```

## NFS volumes and instance directories on NFS

Cinder NFS volumes are mounted by `nova-compute` under `nfs_mount_point_base`, by default `$state_path/mnt`
(`/var/lib/nova/mnt`) ([`nova/conf/libvirt.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/conf/libvirt.py)).
The NFS client logs in the kernel log, from
[`net/sunrpc/clnt.c`](https://github.com/torvalds/linux/blob/v7.3/net/sunrpc/clnt.c):

```text
<program>: server <server> not responding, still trying
<program>: server <server> not responding, timed out
<program>: server <server> OK
```

- `<program>` is the RPC program, `nfs` for NFS itself.
- `still trying` is logged for a request that keeps being retried (a `hard` mount, the default): I/O blocks until the
  server answers, and processes that use the mount, QEMU included, can end up as hung tasks.
- `timed out` is logged for a `soft` request: the I/O fails with an error.
- `OK` marks the server answering again.

```bash
journalctl -k | grep -E "server .* (not responding|OK)"
findmnt -t nfs,nfs4               # what is mounted, with the options (hard/soft, timeo, retrans)
```

## Ceph RBD: the client logs on the compute

With RBD, QEMU opens the volume through librbd inside its own process: there is no kernel block device and no
separate daemon to read. The Ceph client logs nowhere useful by default. Ceph's
[OpenStack guide](https://github.com/ceph/ceph/blob/tentacle/doc/rbd/rbd-openstack.rst) recommends, in the compute's
`ceph.conf`, an admin socket and a log file per QEMU process:

```ini
[client]
    admin socket = /var/run/ceph/guests/$cluster-$type.$id.$pid.$cctid.asok
    log file = /var/log/qemu/qemu-guest-$pid.log
```

with the directories created and owned by the QEMU user (the guide's example is for Red Hat systems). It applies to
QEMU processes started after the change; the guide notes that a running instance needs a restart to get the admin
socket. Changing `ceph.conf` is a configuration change of the compute; on Kolla-Ansible, `ceph.conf` for Nova is a
file that Kolla-Ansible copies into the containers, so make the change in its source and redeploy rather than
editing inside a container.

> ⚠️ **State-changing.** Adds two settings to the Ceph client configuration of the compute. On Kolla-Ansible, the
> redeploy that applies them restarts the Nova containers of that compute (including `nova_libvirt`; see the
> precondition in [restarting nova_libvirt](../lifecycle/restarting-nova-libvirt-does-not-stop-instances.md)). Only
> QEMU processes started afterwards get the socket and the log file: a running instance picks them up only after a
> stop and start (or hard reboot), which interrupts its workload. Each QEMU process then writes its own log file, which
> grows with the client's log level. Undo: remove the two lines, apply the configuration again the same way, and
> delete the files left in the two directories.

**Kolla-Ansible:** QEMU runs inside the `nova_libvirt` container, so `/var/run/ceph/guests/` and `/var/log/qemu/`
must exist **inside that container**, owned by the user QEMU runs as. No volume is mounted at those paths by default
(see `nova_libvirt_default_volumes` in the nova-cell
[defaults](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/nova-cell/defaults/main.yml)),
so the log files are lost when the container is recreated, unless an extra volume is added for them
(`nova_libvirt_extra_volumes`).

When it is in place:

```bash
# On the compute, in the context that runs QEMU (Kolla-Ansible: docker exec nova_libvirt ...)
ls /var/run/ceph/guests/
ceph daemon /var/run/ceph/guests/<socket>.asok help        # what the socket offers
ceph daemon /var/run/ceph/guests/<socket>.asok perf dump   # librbd and objecter counters
less /var/log/qemu/qemu-guest-<qemu-pid>.log
```

Without that configuration, a stalled Ceph cluster shows on the compute only indirectly: the guest's own I/O errors or
hang (its console log), QEMU pausing the instance on I/O errors (see
[the libvirt and QEMU logs](libvirt-and-qemu-logs.md)), and on the Ceph
side, slow requests in the cluster's health.

## A short reading order for "an instance with a volume froze or got I/O errors"

1. Which backend? `openstack volume show <volume-id>` (admin: `os-vol-host-attr:host`, the backend in the host
   string) and `openstack server show <server-id>`.
2. iSCSI: `journalctl -k` for `detected conn error` and `session recovery timed out`; the iscsid output.
3. Multipath: `remaining active paths: 0`, `Entering recovery mode`, `Disable queueing`.
4. NFS: `not responding, still trying` and hung tasks in the kernel log.
5. RBD: the client admin socket and log, if configured; otherwise the Ceph cluster's health.
6. Then the kernel's block errors and the guest's console log.

## Validity and limits

- Written from upstream source and documentation only: open-iscsi 2.1.13, multipath-tools 0.15.1, Linux v7.3, Ceph
  `tentacle` documentation, Kolla-Ansible and Nova `stable/2026.1`. The tables are format strings from that source;
  the iSCSI block marked as captured is from an all-in-one lab (2026-10-27).
- iSCSI without multipath checked on that lab: `journalctl -u kolla-iscsid-container.service`, `docker logs iscsid`,
  `journalctl -k`, `docker exec iscsid iscsiadm -m session -P 1`. Not run: multipath (`multipathd` not deployed there;
  `multipath -ll` on the host needs root), NFS (no NFS backend), the Ceph RBD client logs (no Ceph).
- The exact unit and container names assume Kolla-Ansible's `kolla-<container>-container.service` naming
  (stable/2026.1).
- Fibre Channel, NVMe-oF and other `os-brick` connectors are not covered.
- Where Kolla-Ansible keeps the compute's `ceph.conf` for Nova was not detailed here.
- Written for Kolla-Ansible. The package-install notes (`journalctl -u iscsid`, `journalctl -u multipathd`, the
  kernel lines) apply elsewhere too, but the page was not checked against another deployment method.

---

*Provenance: written by agent:ozarland-author, from upstream documentation and source code (open-iscsi,
multipath-tools, Linux, Ceph, Kolla-Ansible, Nova). Reviewed in its pull request (see the page history). Checked by a
human operator: not yet.*
