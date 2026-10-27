---
title: Reading the libvirt and QEMU logs of a compute, and the guest console log
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [nova, libvirt, qemu]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: any
  storage: any
  os: any
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0 all-in-one (libvirt 10.0.0, QEMU 8.2.2)", date: 2026-10-27, note: "Kolla-Ansible steps on a disposable lab: daemon log, runtime debug with virt-admin and its undo, persistent debug with nova_libvirt_logging_debug and its undo, instance log on start, stop, hard reboot, SIGKILL and SIGTERM of QEMU, console log; not run: the package-install (modular daemon) commands, live migration, KVM errors, panic, OOM killer" }
upstream_refs:
  - https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/docs/kbase/debuglogs.rst
  - https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/docs/daemons.rst
  - https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/src/util/virlog.c
  - https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/src/qemu/qemu_process.c
  - https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/src/qemu/qemu_driver.c
  - https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/src/qemu/qemu_migration.c
  - https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/src/conf/domain_conf.c
  - https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/src/qemu/qemu.conf.in
  - https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/tools/virt-admin.c
  - https://github.com/openstack/kolla/blob/stable/2026.1/docker/nova/nova-libvirt/kolla_nova_libvirt_start.sh
  - https://gitlab.com/qemu-project/qemu/-/blob/v11.1.2/system/runstate.c
  - https://gitlab.com/qemu-project/qemu/-/blob/v11.1.2/accel/kvm/kvm-all.c
  - https://gitlab.com/qemu-project/qemu/-/blob/v11.1.2/util/error-report.c
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/virt/libvirt/driver.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/compute/manager.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/virt/event.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/db/main/api.py
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/nova-cell/templates/libvirtd.conf.j2
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/nova-cell/templates/qemu.conf.j2
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/nova-cell/defaults/main.yml
  - https://github.com/openstack/python-openstackclient/blob/stable/2026.1/openstackclient/compute/v2/console.py
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Reading the libvirt and QEMU logs of a compute, and the guest console log

Under `nova-compute`, three more logs tell what happened to an instance. Each one answers a different question:

| Log | Written by | Answers |
|---|---|---|
| The libvirt daemon log | `libvirtd` (monolithic) or `virtqemud` (modular) | What libvirt was asked to do, and what failed inside libvirt |
| The instance log, `/var/log/libvirt/qemu/<instance-name>.log` | libvirt (header and lifecycle lines) and the QEMU process (its stderr) | When QEMU started, with which command line, and why it stopped |
| The guest console log, `console.log` in the instance directory | The guest operating system, through its serial port | What the guest itself printed: boot, kernel panic, cloud-init |

`<instance-name>` is the libvirt domain name, `instance-<hex id>` by default. Get it with
`openstack server show <server-id> -c OS-EXT-SRV-ATTR:instance_name -c OS-EXT-SRV-ATTR:host` (admin credentials).

## The libvirt daemon log

### Where it is

**Kolla-Ansible:** the `nova_libvirt` container runs the monolithic `libvirtd`. Kolla-Ansible's
[`libvirtd.conf.j2`](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/nova-cell/templates/libvirtd.conf.j2)
writes it to a file on the `kolla_logs` volume, at warning level by default and at debug level when
`nova_libvirt_logging_debug` is true (it follows `nova_logging_debug`):

```text
log_level = 3
log_outputs = "3:file:/var/log/kolla/libvirt/libvirtd.log"
```

```bash
# On the compute (root)
tail -n 100 /var/log/kolla/libvirt/libvirtd.log
grep -E ': (error|warning) : ' /var/log/kolla/libvirt/libvirtd.log | tail -20
```

Each start of `libvirtd` logs one `error` line per QEMU binary of the image that cannot use KVM, about sixty lines
of noise (captured on a lab, Kolla-Ansible 22.2.0):

```text
2026-10-27 07:28:41.691+0000: 4585: error : virQEMUCapsCacheLookupDefault:5959 : invalid argument: the accel 'kvm' is not supported by '/usr/bin/qemu-system-sh4eb' on this host
```

**Any (package install):** first find out whether the host runs the modular daemons or the monolithic one. libvirt's
[daemons documentation](https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/docs/daemons.rst) gives this test:

```bash
# On the compute
systemctl is-active virtqemud.socket virtqemud.service   # either active: modular, QEMU is driven by virtqemud
systemctl is-active libvirtd.socket libvirtd.service     # active and virtqemud inactive: monolithic libvirtd
```

The configuration file is named after the daemon: `/etc/libvirt/virtqemud.conf` or `/etc/libvirt/libvirtd.conf`.
Without `log_outputs`, the default output on a host with journald is `3:journald` (warnings and errors to the journal)
([debug logs kbase](https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/docs/kbase/debuglogs.rst)):

```bash
# On the compute
grep -E '^\s*log_(filters|outputs|level)' /etc/libvirt/virtqemud.conf /etc/libvirt/libvirtd.conf
journalctl -u virtqemud --since "1 hour ago"     # or -u libvirtd
```

### Reading one line

A file output line is built by `virLogOutputToFd` and `virLogFormatString` in
[`src/util/virlog.c`](https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/src/util/virlog.c). Its shape is:

```text
<YYYY-MM-DD HH:MM:SS.mmm+0000>: <thread-id>: <level> : <function>:<line> : <message>
```

A debug line captured on the lab:

```text
2026-10-27 07:31:03.520+0000: 4589: debug : qemuProcessLaunch:7638 : Creating domain log file
```

- The timestamp is always **UTC** (`+0000`), whatever the host's time zone. Check the time zone of the other logs
  you compare it with (`timedatectl` on the host) before you line them up.
- `<level>` is `debug`, `info`, `warning` or `error`.
- `<function>:<line>` points into libvirt's source; `qemuMonitor*` functions talk to QEMU, `virNetSocket*` and
  `virNetClient*` to clients such as `nova-compute`, `qemuMigration*` to live migration.

### Log levels, outputs and filters

From the [debug logs kbase](https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/docs/kbase/debuglogs.rst):

- Levels: `1` debug, `2` info, `3` warning, `4` error.
- `log_outputs` is a space-separated list of `level:output`, where output is `file:<path>`, `journald`, `stderr` or
  `syslog:<name>`. journald can throttle a busy daemon, so debug logs belong in a file.
- `log_filters` is a space-separated list of `level:identifier`, where the identifier is a libvirt module name and
  accepts globs. A filter limits that module to the given level. The kbase gives this set, which keeps everything at
  debug except the noisiest modules:

```text
3:remote 4:event 3:util.json 3:util.object 3:util.dbus 3:util.netlink 3:node_device 3:rpc 3:access 1:*
```

### Turning debug on, then off

> ⚠️ **State-changing.** Debug logging writes far more and can fill the disk that holds the log. The runtime method
> below does not restart the daemon, and the compute's instances keep running. Change one compute at a time, and undo
> it as shown below when you have what you need.

**At runtime, without a restart:** `virt-admin` changes the filters and outputs of the running daemon. The change is
lost when the daemon restarts or stops. The admin URI is `virtqemud:///system` for the modular daemon and
`libvirtd:///system` for the monolithic one, including Kolla-Ansible's. First note the current values: `virt-admin`
prints them as `Logging filters: <filters>` and `Logging outputs: <outputs>`
([`tools/virt-admin.c`](https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/tools/virt-admin.c)).

**Kolla-Ansible:** the `nova_libvirt` container runs `libvirtd --listen` with no idle timeout
([`kolla_nova_libvirt_start.sh`](https://github.com/openstack/kolla/blob/stable/2026.1/docker/nova/nova-libvirt/kolla_nova_libvirt_start.sh)).
Run `virt-admin` inside the container, if the image ships it (`command -v virt-admin`; the 2026.1 Ubuntu image
ships `/usr/bin/virt-admin`); otherwise use the persistent method below. Write the debug file on the `kolla_logs` volume, next to the normal log:

```bash
# On the compute (root). With Podman, use podman exec instead of docker exec.
docker exec nova_libvirt virt-admin -c libvirtd:///system daemon-log-filters    # note: Logging filters: ...
docker exec nova_libvirt virt-admin -c libvirtd:///system daemon-log-outputs    # note: Logging outputs: ...

docker exec nova_libvirt virt-admin -c libvirtd:///system daemon-log-outputs "1:file:/var/log/kolla/libvirt/libvirtd-debug.log 3:file:/var/log/kolla/libvirt/libvirtd.log"
docker exec nova_libvirt virt-admin -c libvirtd:///system daemon-log-filters "3:remote 4:event 3:util.json 3:util.object 3:util.dbus 3:util.netlink 3:node_device 3:rpc 3:access 1:*"
```

On the lab, libvirt 10.0.0 printed the filters back as `3:*remote* 4:*event* ... 1:***`, and one instance start wrote
about 780 KB to the debug file.

**Any, modular daemon:** the daemon is usually socket-activated and stops after an idle timeout (120 seconds in common
deployments, per the kbase), which would discard the runtime settings. `daemon-timeout --timeout 0` (libvirt 8.6.0 and
later) keeps it running:

```bash
# On the compute (root)
virt-admin -c virtqemud:///system daemon-log-filters     # note: Logging filters: ...
virt-admin -c virtqemud:///system daemon-log-outputs     # note: Logging outputs: ...
virt-admin -c virtqemud:///system daemon-timeout --timeout 0

virt-admin -c virtqemud:///system daemon-log-outputs "1:file:/var/log/libvirt/virtqemud-debug.log 3:journald"
virt-admin -c virtqemud:///system daemon-log-filters "3:remote 4:event 3:util.json 3:util.object 3:util.dbus 3:util.netlink 3:node_device 3:rpc 3:access 1:*"
```

For the monolithic `libvirtd` on a package install, use `-c libvirtd:///system` and a file name such as
`/var/log/libvirt/libvirtd-debug.log`.

**Undo:** set back exactly the two values you noted, with the same `daemon-log-outputs "<noted outputs>"` and
`daemon-log-filters "<noted filters>"` commands. For the filters, an empty noted value is restored with
`daemon-log-filters ""`; this is the case on Kolla-Ansible, whose `libvirtd.conf.j2` sets no `log_filters`. For the
outputs, never restore with the empty string: an empty output list falls back to the host's default output, which can
differ from what was configured (for example a different journald level). The `daemon-timeout --timeout 0` setting has no undo command to read back; it ends with the next restart of the
daemon, after which the configured timeout applies again. Delete the debug file when you no longer need it.

**Persistent methods.** Both rewrite the configuration and then restart the libvirt daemon.

> ⚠️ **State-changing.** Restarts the libvirt daemon of the compute. While it restarts, libvirt operations on this
> compute (start, stop, new instances, migrations) wait. Precondition: no migration is in progress to or from the
> compute: `openstack server migration list --host <compute>`, on a controller with admin credentials, shows none
> whose status is other than `confirmed`, `reverted`, `error`, `failed`, `completed`, `cancelled` or `done` (Nova's
> own definition of in progress, `migration_get_in_progress_by_host_and_node`; it covers `accepted`, `queued`,
> `preparing`, `running`, `post-migrating`, and `pre-migrating`, `migrating` and `finished` for cold migrations and
> resizes; filtering on `--status running` alone misses most of them), and on Kolla-Ansible `nova_libvirt` runs in the host's PID and cgroup namespaces (quick check 3 of
> [restarting nova_libvirt](../lifecycle/restarting-nova-libvirt-does-not-stop-instances.md) prints `host host`);
> otherwise every instance of the compute can stop or a migration can fail. Undo: revert the setting and restart again,
> under the same precondition.

**Kolla-Ansible:** set `nova_libvirt_logging_debug: true` in `/etc/kolla/globals.yml` (or in the host's `host_vars`, to
limit it to one compute), then run `kolla-ansible reconfigure -i <inventory> --limit <compute> --tags nova`. This
rewrites `libvirtd.conf` with `log_level = 1` and restarts `nova_libvirt`. Undo by removing the variable and running
the same command again. With no filters, level 1 logs every module: expect a large file. On the lab, each
reconfigure took about 4 minutes; the running instance's QEMU process kept its pid; `libvirtd.log` grew by 2 MB in the first 25 seconds
after the restart. The debug lines go into `libvirtd.log` itself, not a separate file, and stay there after the
undo (10 MB on the lab): leave it to Kolla-Ansible's logrotate (the `cron` container) rather than truncating it, which
loses the lines.

**Any:** set `log_filters` and `log_outputs` in the daemon's configuration file, then restart the socket or the service
(`systemctl restart virtqemud.socket`, or `libvirtd.socket`/`libvirtd.service`), as the kbase describes. Whether the
instances survive that restart on a package install depends on the distribution's unit configuration and was not
checked here.

## The instance log: /var/log/libvirt/qemu/\<instance-name\>.log

### Where it is

libvirt opens this file for every QEMU process it starts and appends both its own lines and QEMU's stderr. Kolla-Ansible
sets `stdio_handler = "logd"` in
[`qemu.conf.j2`](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/nova-cell/templates/qemu.conf.j2),
which is also libvirt's default: the `virtlogd` daemon writes and rotates the file, so a running instance's log
survives a restart of `libvirtd` alone. On Kolla-Ansible, `virtlogd` runs in the same `nova_libvirt` container
(`kolla_nova_libvirt_start.sh`), so restarting the container restarts `virtlogd` too. On the lab, a restart of
`nova_libvirt` (by a `reconfigure`) kept the instance log file; whether the running instance's later lines still
reach it was not checked.

```bash
# Any (package install), on the compute
less /var/log/libvirt/qemu/<instance-name>.log

# Kolla-Ansible: the directory is inside the nova_libvirt container, not on the kolla_logs volume
docker exec nova_libvirt ls -l /var/log/libvirt/qemu/
docker exec nova_libvirt tail -n 50 /var/log/libvirt/qemu/<instance-name>.log
```

The file is per instance and per host: after a live migration, the start on the destination is in the destination's
file, and the source's file ends with the migration.

**Kolla-Ansible:** no volume is mounted at `/var/log/libvirt` in `nova_libvirt` (see `nova_libvirt_default_volumes` in
the nova-cell [defaults](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/nova-cell/defaults/main.yml)).
The instance logs live in the container's own layer and are **lost when `nova_libvirt` is recreated**, for example on an
upgrade or a redeploy. Copy the file out (`docker cp`, or `podman cp`) before such an operation if you still need it.

### The lines libvirt writes, and what they mean

All the lines below are format strings from libvirt's source (v12.8.0). libvirt timestamps them in UTC. Lines
captured on a lab (Kolla-Ansible 22.2.0, whose Ubuntu image ships libvirt 10.0.0 and QEMU 8.2.2), with the action
that wrote them:

```text
2026-10-27 07:31:03.698+0000: starting up libvirt version: 10.0.0, package: 10.0.0-2ubuntu8.19 (Ubuntu), qemu version: 8.2.2Debian 1:8.2.2+ds-0ubuntu1.18, kernel: 6.8.0-100-generic, hostname: compute-1
char device redirected to /dev/pts/0 (label charserial0)
2026-10-27T07:32:04.145625Z qemu-system-x86_64: terminating on signal 15 from pid 3513 (/usr/sbin/libvirtd)
2026-10-27 07:32:04.380+0000: shutting down, reason=shutdown          # openstack server stop, guest answered ACPI
2026-10-27 07:33:08.635+0000: shutting down, reason=crashed           # kill -9 of QEMU from the host: no QEMU line
2026-10-27T07:34:51.263738Z qemu-system-x86_64: terminating on signal 15 from pid 3513 (/usr/sbin/libvirtd)
2026-10-27 07:34:51.464+0000: shutting down, reason=destroyed         # openstack server reboot --hard
2026-10-27T07:35:28.197568Z qemu-system-x86_64: terminating on signal 15 from pid 27070 ()
2026-10-27 07:35:28.458+0000: shutting down, reason=shutdown          # kill -15 of QEMU from the host
```

| Line (format) | Written when | Source |
|---|---|---|
| `<timestamp>: starting up libvirt version: <version>, qemu version: <x.y.z>, kernel: <release>, hostname: <host>` followed by the full QEMU command line | Each QEMU start: boot, hard reboot, power on, incoming migration, unshelve | `qemuLogOperation` in `qemu_process.c` |
| `<timestamp>: initiating migration` (or `resuming migration`) | On the **source**, when libvirt asks QEMU to start migrating | `qemu_migration.c` |
| `<timestamp>: shutting down, reason=<reason>` | Each stop of the QEMU process, written once the process has been killed or has exited, while libvirt cleans up | `qemuProcessStop` in `qemu_process.c` |
| `<timestamp>: panic <details>` | The guest reported a panic through a pvpanic or Hyper-V crash device, when the instance has one | `processGuestPanicEvent` in `qemu_driver.c` |

A command line that contains `-incoming` belongs to the destination side of a migration: QEMU starts there waiting for
the incoming state.

### The shutdown reason

`<reason>` is the libvirt shutoff reason (`virDomainShutoffReason` in `domain_conf.c`). The values that matter on a
compute, with the code path that sets them:

| reason | What happened | Typical OpenStack cause |
|---|---|---|
| `shutdown` | QEMU reported a shutdown before it exited: the guest's own, or QEMU's on a SIGTERM from outside libvirt (the lab: `kill -15` gave `reason=shutdown`, after `terminating on signal 15 from pid <n> ()`) | The guest powered itself off, or answered the ACPI request of a `openstack server stop` within `shutdown_timeout` (60 s by default) |
| `destroyed` | libvirt was asked to kill the domain | `openstack server stop` after the guest ignored the shutdown request, hard reboot, delete, or the destroy that ends every Nova power off |
| `crashed` | QEMU exited **without** a shutdown event (libvirt's debug log says it "disappeared without SHUTDOWN event; assuming the domain crashed") | QEMU killed from outside (for example by the kernel OOM killer), or QEMU aborted. Read the kernel log of that minute next |
| `migrated` | The migration completed and the source QEMU was stopped | Live migration, on the source |
| `failed` | Start or incoming migration failed and libvirt cleaned up | Failed boot on this host, or the destination side of a failed migration |
| `saved`, `from snapshot`, `daemon` | Managed save, snapshot revert, daemon-driven stop | Rare with Nova |

Nova's `power_off` first asks the guest to shut down (`_clean_shutdown`, which retries the request and waits
`shutdown_timeout`), then always calls destroy
([`driver.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/virt/libvirt/driver.py)). The reason tells you
which of the two actually stopped the guest. `nova-compute.log` has the matching lines
`Instance shutdown successfully after <n> seconds.` or `Instance failed to shutdown in <n> seconds.`

### The lines QEMU writes

Messages that QEMU reports through `error_report` carry its program name and, because libvirt starts it with
`-msg timestamp=on`, an ISO 8601 UTC timestamp
([`util/error-report.c`](https://gitlab.com/qemu-project/qemu/-/blob/v11.1.2/util/error-report.c)). Messages that the
KVM code writes straight to stderr with `fprintf` have **no** prefix and no timestamp: place them by the libvirt lines
around them. Messages worth knowing (formats from QEMU v11.1.2):

| Message (format) | Meaning | Source |
|---|---|---|
| `terminating on signal <n> from pid <pid> (<process name>)`, or `terminating on signal <n>` when the sender is unknown | QEMU received a catchable signal (15 is SIGTERM) and exits; the pid and name say who sent it. libvirt sends SIGTERM to stop QEMU, so this line naming the libvirt daemon, just **before** `shutting down, reason=destroyed` or `reason=shutdown` (the lab showed it for both a hard reboot and a clean `server stop`), is normal. Another sender is worth investigating; an empty name `()` means the sender had already exited | `qemu_kill_report` in `system/runstate.c` |
| `KVM internal error. Suberror: <n>` (no prefix), followed by `extra data[<i>]: ...` and, for suberror 1, `emulation failure` and a register dump | KVM could not handle a guest exit. The guest stops; check the kernel log of the host for KVM messages at the same time | `kvm_handle_internal_error` in `accel/kvm/kvm-all.c` |
| `error: kvm run failed <errno text>` (no prefix) | The `KVM_RUN` call itself failed | `kvm_cpu_exec` in `accel/kvm/kvm-all.c` |

A SIGKILL cannot be caught: when the kernel OOM killer kills QEMU, QEMU writes nothing. The instance log then only
shows `shutting down, reason=crashed`, and the explanation is in `journalctl -k` on the host: the kernel's
`oom_kill_process` logs `Out of memory: Killed process <pid> (<command>) total-vm:...`, where `<command>` is the QEMU
binary name ([`mm/oom_kill.c`](https://github.com/torvalds/linux/blob/master/mm/oom_kill.c)).

### What Nova logs at the same moment

When libvirt reports a lifecycle change, `nova-compute.log` logs `VM <state> (Lifecycle Event)` with the instance uuid,
where `<state>` is one of `Started`, `Stopped`, `Paused`, `Resumed`, `Suspended`, `Postcopy started`,
`Migration completed` ([`nova/virt/event.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/virt/event.py),
`handle_lifecycle_event` in [`manager.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/compute/manager.py)).
If the instance was supposed to be running, the power state sync then logs
`Instance shutdown by itself. Calling the stop API.` and stops it in Nova's database: Nova does not restart an instance
that stopped on its own. On the lab, after QEMU was killed with SIGKILL (`reason=crashed` at 07:33:08.6):

```text
2026-10-27 07:33:23.653 6 INFO nova.compute.manager [-] [instance: <server-id>] VM Stopped (Lifecycle Event)
2026-10-27 07:33:24.676 6 INFO nova.compute.manager [...] [instance: <server-id>] During _sync_instance_power_state the DB power_state (1) does not match the vm_power_state from the hypervisor (4). Updating power_state in the DB to match the hypervisor.
2026-10-27 07:33:25.188 6 WARNING nova.compute.manager [...] [instance: <server-id>] Instance shutdown by itself. Calling the stop API. Current vm_state: active, current task_state: None, original DB power_state: 1, current VM power_state: 4
```

The lifecycle event came 15 seconds after the stop.

```bash
# On the compute: the Nova side, then the libvirt side, of one instance
grep "<server-id>" /var/log/kolla/nova/nova-compute.log | grep -E "Lifecycle Event|shutdown by itself|failed to shutdown"
docker exec nova_libvirt grep -E "starting up|shutting down|initiating migration|terminating on signal|KVM internal error|panic" \
  /var/log/libvirt/qemu/<instance-name>.log
```

## The guest console log

### What it contains

For the `kvm` and `qemu` virt types, Nova gives each instance a serial device whose output libvirt logs to
`console.log` in the instance directory, `<instances_path>/<server-id>/console.log` (by default
`/var/lib/nova/instances/<server-id>/console.log`) (`_create_consoles_qemu_kvm` and `_get_console_log_path` in
[`driver.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/virt/libvirt/driver.py)). It holds only what the
guest writes to its first serial port: a guest whose kernel and init do not log to the serial console leaves it empty.

### Reading it

```bash
# From anywhere with the project's credentials (or admin)
openstack console log show <server-id>
openstack console log show --lines 50 <server-id>    # only the last 50 lines
```

Nova returns at most the last 100 KiB (`MAX_CONSOLE_BYTES`), reading `console.log` and then the rotated
`console.log.0`, `console.log.1`, … if needed; when it cuts, `nova-compute.log` says
`Truncated console log returned, <n> bytes ignored`. An instance without any console raises `ConsoleNotAvailable`; a
console whose file does not exist yet returns an empty output and logs `console logfile for instance does not exist`.

```bash
# On the compute, the same file directly (root). Kolla-Ansible: the directory is in the nova_compute volume;
# console.log is root-only (mode 600) and docker exec runs as the nova user, hence -u root
docker exec -u root nova_compute tail -n 50 /var/lib/nova/instances/<server-id>/console.log
```

### What to look for

| You see in the console log | It points to |
|---|---|
| Nothing at all, while the instance log shows `starting up` and no stop | The guest does not log to the serial console, or it hangs before its kernel starts (firmware, boot disk). Compare with the VNC/SPICE console |
| A kernel panic or an `Oops` | A guest problem (image, kernel, driver), not an OpenStack one, unless it follows a host-side event at the same time |
| cloud-init errors fetching metadata | The metadata path: see [instance boots without metadata](../networking/new-instance-boots-without-metadata.md) |
| No DHCP lease | See [instance never got an IP address](../networking/new-instance-gets-no-ip-dhcp-agents-down.md) |

## A short reading order for "my instance stopped"

1. `openstack server event list <server-id>`: did someone (or Nova) ask for a stop, reboot, migration or delete?
2. The instance log on the host that ran it: the last `shutting down, reason=` line and the QEMU messages just before.
3. If `reason=crashed`: `journalctl -k` on that host around the same UTC minute (OOM killer, KVM errors, hung tasks).
4. If `reason=shutdown` with no API action: the guest powered itself off; read the console log.
5. `nova-compute.log` for the lifecycle event and the power state sync that followed.

Related runbooks: [compute disk full, instance paused](../lifecycle/compute-disk-full-instance-paused.md),
[live migration never completes](../migration/live-migration-never-completes-or-times-out.md),
[after a compute reboot](../recovery/after-a-compute-reboot-instances-shutoff-or-without-network.md).

## Lab observations

On an all-in-one lab (OpenStack 2026.1, Kolla-Ansible 22.2.0, Ubuntu 24.04, Docker, the Kolla Ubuntu image with
libvirt 10.0.0 and QEMU 8.2.2, a cirros 0.6.3 instance):

| Step | Observed |
|---|---|
| Daemon log, `tail` and `grep` | As written; about 60 `accel 'kvm' is not supported by '/usr/bin/qemu-system-<arch>'` errors at each `libvirtd` start |
| Runtime debug with `virt-admin`, then the undo | As written; `virt-admin` is in the image; 780 KB of debug for one instance start; after the undo the debug file stopped growing |
| Persistent debug, `nova_libvirt_logging_debug: true` and `reconfigure --tags nova`, then the undo | As written, `failed=0`, 4 min 33 s and 3 min 47 s; the instance kept running (same QEMU pid); the debug lines stay in `libvirtd.log` |
| Instance log on start, `server stop`, hard reboot, `kill -9` and `kill -15` of QEMU | The lines quoted above. A clean stop also shows `terminating on signal 15` from `libvirtd`; an outside SIGTERM gives `reason=shutdown`; SIGKILL gives `reason=crashed` with no QEMU line |
| Nova side | `VM Stopped (Lifecycle Event)` 15 s after the crash, then `Instance shutdown by itself. Calling the stop API.`; `Instance shutdown successfully after 3 seconds.` for the clean stop (quoted from the `nova-compute.log` capture) |
| Console log | `openstack console log show` as written; the `docker exec nova_compute tail` failed with `Permission denied` and needed `-u root` |

Not run: the package-install commands (modular `virtqemud`), a live migration (one node), a guest that ignores ACPI,
KVM internal errors, a guest panic, the OOM killer, a recreate of `nova_libvirt`.

## Validity and limits

- Written from upstream source and documentation: libvirt v12.8.0, QEMU v11.1.2, Nova, Kolla-Ansible and
  python-openstackclient `stable/2026.1`. The lines marked as captured come from a Kolla-Ansible 22.2.0 lab, whose
  image ships libvirt 10.0.0 and QEMU 8.2.2; the other messages are format strings from the source. Releases may word
  some lines differently; check your versions with the `starting up` header.
- The migration, KVM error, panic and OOM lines are not captured yet (see *Lab observations*).
- The OOM killer line is quoted from the Linux kernel's current source; distribution kernels can differ. The kernel
  logs are covered on their own page.
- `virt-admin` is not part of every distribution's libvirt client package. The Kolla 2026.1 Ubuntu image ships it;
  other base distributions not checked.
- The `panic` line needs a pvpanic or Hyper-V crash device in the domain. Whether Nova adds one for a given image
  was not checked; most instances never produce this line.

---

*Provenance: written by agent:ozarland-author, from upstream documentation and source code (libvirt, QEMU, Nova,
Kolla-Ansible). Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*
