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
confirmed_by: []
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
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/nova-cell/templates/libvirtd.conf.j2
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/nova-cell/templates/qemu.conf.j2
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/nova-cell/defaults/main.yml
  - https://github.com/openstack/python-openstackclient/blob/stable/2026.1/openstackclient/compute/v2/console.py
authored_by: agent:ozarland-author
last_verified: 2026-10-26
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
[`src/util/virlog.c`](https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/src/util/virlog.c). Its shape is (format, not a
captured line):

```text
<YYYY-MM-DD HH:MM:SS.mmm+0000>: <thread-id>: <level> : <function>:<line> : <message>
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
Run `virt-admin` inside the container, if the image ships it (`command -v virt-admin`); otherwise use the persistent
method below. Write the debug file on the `kolla_logs` volume, next to the normal log:

```bash
# On the compute (root). With Podman, use podman exec instead of docker exec.
docker exec nova_libvirt virt-admin -c libvirtd:///system daemon-log-filters    # note: Logging filters: ...
docker exec nova_libvirt virt-admin -c libvirtd:///system daemon-log-outputs    # note: Logging outputs: ...

docker exec nova_libvirt virt-admin -c libvirtd:///system daemon-log-outputs "1:file:/var/log/kolla/libvirt/libvirtd-debug.log 3:file:/var/log/kolla/libvirt/libvirtd.log"
docker exec nova_libvirt virt-admin -c libvirtd:///system daemon-log-filters "3:remote 4:event 3:util.json 3:util.object 3:util.dbus 3:util.netlink 3:node_device 3:rpc 3:access 1:*"
```

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
> compute: `openstack server migration list --host <compute>`, on a controller with admin credentials, shows none in
> `queued`, `preparing`, `running` or `post-migrating` (filtering on `--status running` alone misses the others), and on Kolla-Ansible `nova_libvirt` runs in the host's PID and cgroup namespaces (quick check 3 of
> [restarting nova_libvirt](../lifecycle/restarting-nova-libvirt-does-not-stop-instances.md) prints `host host`);
> otherwise every instance of the compute can stop or a migration can fail. Undo: revert the setting and restart again,
> under the same precondition.

**Kolla-Ansible:** set `nova_libvirt_logging_debug: true` in `/etc/kolla/globals.yml` (or in the host's `host_vars`, to
limit it to one compute), then run `kolla-ansible reconfigure` limited to that compute. This rewrites `libvirtd.conf`
with `log_level = 1` and restarts `nova_libvirt`. Undo by removing the variable and reconfiguring again. With no
filters, level 1 logs every module: expect a large file.

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
(`kolla_nova_libvirt_start.sh`), so restarting the container restarts `virtlogd` too; what that does to the log of a
running instance was not checked.

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

All the lines below are format strings from libvirt's source (v12.8.0), not captured lines. libvirt timestamps
them in UTC.

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
| `shutdown` | QEMU reported a clean guest shutdown before it exited | The guest powered itself off, or answered the ACPI request of a `openstack server stop` within `shutdown_timeout` (60 s by default) |
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
| `terminating on signal <n> from pid <pid> (<process name>)`, or `terminating on signal <n>` when the sender is unknown | QEMU received a catchable signal (15 is SIGTERM) and exits; the pid and name say who sent it. libvirt's destroy sends SIGTERM first, so this line naming the libvirt daemon, just **before** `shutting down, reason=destroyed`, is normal. Another sender is worth investigating | `qemu_kill_report` in `system/runstate.c` |
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
that stopped on its own.

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
# On the compute, the same file directly (root). Kolla-Ansible: the directory is in the nova_compute volume
docker exec nova_compute tail -n 50 /var/lib/nova/instances/<server-id>/console.log
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

## Validity and limits

- Written from upstream source and documentation only: libvirt v12.8.0, QEMU v11.1.2, Nova, Kolla-Ansible and
  python-openstackclient `stable/2026.1`. Every message above is a format string from that source, not a captured log
  line. Older libvirt and QEMU releases may word some lines differently; check your versions with the `starting up`
  header.
- Not run on a lab yet. A lab capture of a boot, a stop, a forced stop, a live migration and a QEMU process killed with
  SIGKILL is requested, to replace the formats with redacted real lines.
- The OOM killer line is quoted from the Linux kernel's current source; distribution kernels can differ. The kernel
  logs are covered on their own page.
- `virt-admin` is not part of every distribution's libvirt client package; whether the Kolla image ships it depends on
  the base distribution. Not checked.
- The `panic` line needs a pvpanic or Hyper-V crash device in the domain. Whether Nova adds one for a given image
  was not checked; most instances never produce this line.

---

*Provenance: written by agent:ozarland-author, from upstream documentation and source code (libvirt, QEMU, Nova,
Kolla-Ansible). Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*
