---
title: Reading the systemd journal and the container engine on an OpenStack host. Units, priorities, time ranges, restart loops
kind: diagnostic
status: lab-verified
openstack_releases: ["2026.1"]
components: [systemd, docker, podman, nova, neutron]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: any
  storage: any
  os: any linux with systemd
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-27, note: "systemd 255, Docker 29.9.2, unit with RestartSec=59/StartLimitInterval=600 (docker_client_timeout 600): nova_compute restart loop for 13 min (restart counter to 11, no start-limit hit), NRestarts and RestartCount 0 confirmed, Docker debug via SIGHUP (undo needs debug false), journal commands; not run: Podman, the persistent-journal drop-in, the Kolla default restart values" }
upstream_refs:
  - https://github.com/systemd/systemd/blob/v259.10/man/journalctl.xml
  - https://github.com/systemd/systemd/blob/v259.10/man/journald.conf.xml
  - https://github.com/systemd/systemd/blob/v259.10/src/core/service.c
  - https://github.com/systemd/systemd/blob/v259.10/src/core/unit.c
  - https://github.com/systemd/systemd/blob/v259.10/src/core/job.c
  - https://github.com/systemd/systemd/blob/v259.10/src/journal/journald-manager.c
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/module_utils/kolla_systemd_worker.py
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/module_utils/kolla_docker_worker.py
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/module_utils/kolla_podman_worker.py
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/group_vars/all/common.yml
  - https://github.com/openstack/ansible-collection-kolla/blob/stable/2026.1/roles/docker/vars/main.yml
  - https://github.com/openstack/ansible-collection-kolla/blob/stable/2026.1/roles/podman/defaults/main.yml
  - https://github.com/docker/docs/blob/main/content/manuals/engine/daemon/logs.md
  - https://github.com/docker/docs/blob/main/content/manuals/engine/logging/configure.md
  - https://github.com/docker/docs/blob/main/content/manuals/engine/logging/drivers/journald.md
  - https://github.com/moby/moby/blob/v28.5.2/api/swagger.yaml
  - https://github.com/moby/moby/blob/v28.5.2/daemon/monitor.go
  - https://github.com/containers/common/blob/main/docs/containers.conf.5.md
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Reading the systemd journal and the container engine on an OpenStack host. Units, priorities, time ranges, restart loops

On a host that runs OpenStack, three things outside the service logs explain many failures: the systemd units that
start the services, the journal that records what those units and the kernel said, and, on a containerised deployment,
the container engine. This page shows how to read each, and how to recognise a service that keeps restarting.

The kernel part of the journal (`journalctl -k`) has its own page:
[reading the kernel log of a compute](kernel-messages-on-computes.md).

## journalctl: the commands that matter

All commands in this section are read-only. Run them on the host, as root or as a member of the `systemd-journal` (or
`adm`) group. Options from [`journalctl(1)`](https://github.com/systemd/systemd/blob/v259.10/man/journalctl.xml),
systemd v259:

```bash
journalctl -u <unit>                       # one unit; -u can be repeated, and accepts patterns
journalctl -u <unit> -f                    # follow
journalctl -u <unit> -n 200 --no-pager     # last 200 lines
journalctl -u <unit> -b                    # current boot only; -b -1 for the previous boot
journalctl --since "2026-10-26 14:00" --until "2026-10-26 14:30"
journalctl --since "-15min"                # relative time; also "today", "yesterday"
journalctl -p err                          # err and more severe (crit, alert, emerg)
journalctl -p warning..err                 # a range, both ends included
journalctl -o short-iso-precise            # ISO 8601 timestamps with microseconds
journalctl -o json-pretty -u <unit> -n 5   # every field of an entry
journalctl -x -u <unit>                    # add the explanation texts of the message catalog
```

- Priorities are the syslog levels: `emerg` (0), `alert` (1), `crit` (2), `err` (3), `warning` (4), `notice` (5),
  `info` (6), `debug` (7). A single level means "this level and more important".
- `--since` and `--until` take `YYYY-MM-DD HH:MM:SS` in the **host's local time zone**. When you line the journal up
  with logs written in UTC (Open vSwitch, libvirt, QEMU), use `-o short-iso-precise`, which prints the offset, or set
  `TZ=UTC` for the command.
- `journalctl --list-boots` lists the boots the journal still holds; `-b -1` fails if the journal is not persistent
  (see below).
- `journalctl -u <unit> --list-invocations` (in recent systemd, documented in v259) lists the successive runs of a
  unit, which is a quick way to count restarts.

## Units on a Kolla-Ansible host

Kolla-Ansible creates one systemd unit per container, named `kolla-<container>-container.service`
([`kolla_systemd_worker.py`](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/module_utils/kolla_systemd_worker.py)).
The unit template, at `stable/2026.1`, is:

```text
[Unit]
...
StartLimitInterval=${restart_timeout}
StartLimitBurst=${restart_retries}

[Service]
ExecStart=/usr/bin/${engine} start -a ${name}
ExecStop=/usr/bin/${engine} stop ${name} -t ${graceful_timeout}
Restart=${restart_policy}
RestartSec=${restart_duration}
SuccessExitStatus=143
```

What follows from it:

- `ExecStart` runs `docker start -a` (or `podman start -a`): the container's standard output and error are attached
  to the unit, so **`journalctl -u kolla-<container>-container.service` shows what the container printed**, including
  the start-up of `kolla_set_configs` and a crash before the service opened its log file.
- `Restart=always` for containers whose restart policy is `unless-stopped`, Kolla-Ansible's default
  (`docker_restart_policy` in
  [`group_vars/all/common.yml`](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/group_vars/all/common.yml)).
- The restart limits come from the client timeout and the retries: with the defaults (120 s and 10 retries),
  `StartLimitInterval=120`, `StartLimitBurst=10` and `RestartSec=11`. A lab with `docker_client_timeout: 600` had
  `StartLimitInterval=600` and `RestartSec=59` (11 s with the Kolla defaults, 59 s on that lab).
- **The container engine does not restart these containers.** The Docker worker creates the container without a
  Docker restart policy, and the Podman worker removes it ("handled by systemd")
  ([`kolla_docker_worker.py`](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/module_utils/kolla_docker_worker.py),
  [`kolla_podman_worker.py`](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/module_utils/kolla_podman_worker.py)).
  This matters for counting restarts: see below.

```bash
# On a Kolla-Ansible host
systemctl list-units 'kolla-*' --no-pager
systemctl status kolla-nova_compute-container.service
journalctl -u kolla-nova_compute-container.service --since "-1h"
```

**Any (package install):** the services are units of the distribution's packages (for example
`nova-compute.service` or `openstack-nova-compute.service`, depending on the distribution). `systemctl list-units
'*nova*' '*neutron*'` finds them; `systemctl cat <unit>` shows the command line and its `--log-file` or
`--config-file`.

## Recognising a restart loop

### What systemd logs

When a service exits and its unit restarts it, systemd logs, for the unit (format strings from
[`service.c`](https://github.com/systemd/systemd/blob/v259.10/src/core/service.c),
[`unit.c`](https://github.com/systemd/systemd/blob/v259.10/src/core/unit.c) and
[`job.c`](https://github.com/systemd/systemd/blob/v259.10/src/core/job.c), not captured lines):

```text
<unit>: Failed with result '<result>'.
<unit>: Scheduled restart job, restart counter is at <n>.
<unit>: Start request repeated too quickly.
Failed to start <unit description>.
```

- `restart counter is at <n>` growing is the loop itself.
- `Start request repeated too quickly.` means the unit hit `StartLimitBurst` starts within `StartLimitInterval`:
  systemd **stops trying**, and the unit stays failed (result `start-limit-hit`) until someone starts it again or
  resets it. On Kolla-Ansible, with the defaults, that is 10 starts in 120 seconds. Because of `RestartSec`, a
  container that fails a few seconds after each start may never reach that rate: on a lab with `RestartSec=59` and
  `StartLimitInterval=600`, a `nova_compute` that exited about 5 s after each start looped for 13 minutes
  (`restart counter is at 11` and growing) without `Start request repeated too quickly.`. Do not wait for the limit:
  read the counter.
- The reason of each exit is in the lines just before, from the service itself: read them with
  `journalctl -u <unit> -n 100`, and the service's own log file.

```bash
# On the host
systemctl show -p NRestarts -p ActiveState -p SubState -p Result kolla-<container>-container.service
journalctl -u kolla-<container>-container.service | grep -E "restart counter is at|repeated too quickly|Failed with result"
```

### What the container engine shows

The Docker API documents, for `docker inspect`
([`swagger.yaml`](https://github.com/moby/moby/blob/v28.5.2/api/swagger.yaml)):

| Field | Meaning |
|---|---|
| `.RestartCount` | Number of times the container was restarted since it was created, or since the daemon was started |
| `.State.Restarting` | Whether the container is restarting |
| `.State.OOMKilled` | A process of the container was killed because it ran out of memory since the container was last started |
| `.State.ExitCode` | The last exit code |
| `.State.StartedAt`, `.State.FinishedAt` | When the container last started and last exited |

`RestartCount` grows only when **Docker's own restart policy** restarts the container (`daemon/monitor.go`
increments it when the restart manager decides to restart). On Kolla-Ansible, systemd restarts the container with
`docker start`, so `RestartCount` stays at `0` and `docker ps` shows `Up <n> seconds` again and again. Use
`StartedAt` and systemd's `NRestarts` instead. Confirmed on a lab: during 11 systemd restarts of `nova_compute`,
`RestartCount` stayed `0` while `StartedAt` changed at each start, and the container's Docker restart policy was `no`.

```bash
# On the host. Podman: same commands with podman instead of docker (Podman not run on the lab).
docker ps -a --format 'table {{.Names}}\t{{.Status}}' | grep -v ' Up [0-9]* \(hours\|days\|weeks\)'
docker inspect -f '{{.Name}} restarts={{.RestartCount}} exit={{.State.ExitCode}} oom={{.State.OOMKilled}} started={{.State.StartedAt}} finished={{.State.FinishedAt}}' <container>
docker logs --since 30m --tail 100 <container>
```

`ExitCode` 137 is 128 + 9 (SIGKILL): the container was killed, by the OOM killer or by a stop that ran out of grace
time. `143` is 128 + 15 (SIGTERM), a normal stop, which the Kolla unit counts as success (`SuccessExitStatus=143`).

## The container engine's own logs

**Docker:** the daemon logs to the journal under `docker.service`
([daemon logs](https://github.com/docker/docs/blob/main/content/manuals/engine/daemon/logs.md)):

```bash
journalctl -xu docker.service --since "-1h"
```

The default log driver for containers is `json-file`
([logging configuration](https://github.com/docker/docs/blob/main/content/manuals/engine/logging/configure.md)).
Kolla-Ansible's host bootstrap sets its rotation to 5 files of 50 MB (`docker_log_max_file`, `docker_log_max_size`
in [ansible-collection-kolla](https://github.com/openstack/ansible-collection-kolla/blob/stable/2026.1/roles/docker/vars/main.yml)),
so `docker logs` only keeps the recent output. Containers that use the `journald` driver instead can be read with
`journalctl CONTAINER_NAME=<name>`
([journald driver](https://github.com/docker/docs/blob/main/content/manuals/engine/logging/drivers/journald.md)).

To get more detail from the daemon itself:

> ⚠️ **State-changing.** Makes `dockerd` reload its configuration and log at debug level, which is verbose. Running
> containers are not restarted by a reload. Undo: set `"debug": false` in `/etc/docker/daemon.json` and send
> `SIGHUP` again (removing the key is not enough: on a lab with Docker 29.9.2, the reload kept debug on; once off,
> the key can be removed). Do not restart `docker.service` for this: unless Docker's `live-restore` is enabled, a daemon
> restart stops every container of the host.

```bash
# On the host: add "debug": true to /etc/docker/daemon.json (keep the existing keys), then:
kill -SIGHUP $(pidof dockerd)
```

If `dockerd` hangs, `kill -SIGUSR1 $(pidof dockerd)` logs a stack trace of all its goroutines without stopping it
(same documentation).

**Podman** *(not run on the lab)*: there is no daemon. With Kolla-Ansible, the host bootstrap sets the `k8s-file` log driver with a 50 MB
maximum (`podman_log_max_size`) and `events_logger: "file"`
([ansible-collection-kolla podman defaults](https://github.com/openstack/ansible-collection-kolla/blob/stable/2026.1/roles/podman/defaults/main.yml));
upstream, Podman uses `journald` by default when the journal is writable
([`containers.conf(5)`](https://github.com/containers/common/blob/main/docs/containers.conf.5.md)). `podman logs`
and `podman inspect` work as with Docker; the unit's journal is the same `kolla-<container>-container.service`.

## Keeping the journal across reboots

From [`journald.conf(5)`](https://github.com/systemd/systemd/blob/v259.10/man/journald.conf.xml):

- `Storage=` decides where the journal lives. `volatile` keeps it in memory (`/run/log/journal`), lost at reboot;
  `persistent` writes it to `/var/log/journal`; `auto` behaves like `persistent` if `/var/log/journal` exists, and
  like `volatile` otherwise. The compiled-in default is distribution-specific.
- `SystemMaxUse=` and `SystemKeepFree=` cap the disk used: by default 10 % and 15 % of the file system, each capped
  at 4G. Only archived files are deleted to stay under the limit.
- `RateLimitIntervalSec=` and `RateLimitBurst=` drop messages from a service that logs more than 10000 messages in
  30 s (the defaults, scaled up with free disk space). The journal then logs `Suppressed <n> messages from <unit>`
  ([`journald-manager.c`](https://github.com/systemd/systemd/blob/v259.10/src/journal/journald-manager.c)): what
  you read around that line is incomplete.
- Configuration goes in `/etc/systemd/journald.conf` or, better, a drop-in `/etc/systemd/journald.conf.d/*.conf`.

```bash
# On the host (read-only): is the journal persistent, and how big is it?
ls -d /var/log/journal 2>/dev/null && echo persistent-directory-present
journalctl --disk-usage
journalctl --list-boots | tail -3
```

> ⚠️ **State-changing.** Makes the journal persistent: it uses disk space under `/var/log/journal`, up to
> `SystemMaxUse`. Restarting `systemd-journald` does not stop the services. Precondition: enough free space on the file
> system that holds `/var/log`. Undo: remove the drop-in, restart `systemd-journald`, and delete `/var/log/journal`
> if you no longer want the stored logs.

```bash
# On the host (not run on the lab)
mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nStorage=persistent\n' > /etc/systemd/journald.conf.d/persistent.conf
systemctl restart systemd-journald
journalctl --flush        # move what is in /run/log/journal to /var/log/journal
```

Kolla-Ansible's fluentd, when enabled, also reads the journal from `/var/log/journal`: without a persistent journal,
it has nothing to read there.

## A short reading order for "this service keeps failing"

1. `systemctl status <unit>`: is it `activating (auto-restart)`, `failed` with `start-limit-hit`, or running?
2. `journalctl -u <unit> -n 200`: the last exit and the lines before it.
3. On Kolla-Ansible: `docker inspect` for `ExitCode`, `OOMKilled` and `StartedAt`; `RestartCount` will not help.
4. The service's own log file, then `journalctl -k` for an OOM kill at the same time.

## Lab observations

Kolla-Ansible 22.2.0 (2026.1) all-in-one, Ubuntu 24.04, systemd 255, Docker 29.9.2, `docker_client_timeout: 600`, on
2026-10-27. A line that is not valid INI appended to `/etc/kolla/nova-compute/nova.conf`, then the unit restarted:

```text
compute-1 docker[<pid>]:     raise ConfigFileParseError(pe.filename, str(pe))
compute-1 systemd[1]: kolla-nova_compute-container.service: Main process exited, code=exited, status=1/FAILURE
compute-1 systemd[1]: kolla-nova_compute-container.service: Failed with result 'exit-code'.
compute-1 systemd[1]: kolla-nova_compute-container.service: Scheduled restart job, restart counter is at 1.
```

- `systemctl show`: `Result=exit-code NRestarts=<n> ActiveState=activating SubState=auto-restart` between starts;
  `docker ps -a`: `Exited (1) 3 seconds ago`; `docker inspect`: `restarts=0 exit=1 oom=false`.
- No `Start request repeated too quickly.` in 13 minutes (one start every 64 s or so, see above). Undo: the file put
  back, `systemctl reset-failed` and `systemctl start`.
- `journalctl --list-invocations`: `unrecognized option` with systemd 255. `journalctl` worked without `sudo` for a
  user in the `adm` group. `/var/log/journal` existed (persistent journal).
- Docker debug: `Got signal to reload configuration`, `Reloaded configuration`, then `level=debug` lines (433 lines
  in 40 s, mostly health checks); removing the key and reloading left debug on, `"debug": false` turned it off.

## Validity and limits

- Written from upstream source and documentation: systemd v259.10, Kolla-Ansible and ansible-collection-kolla
  `stable/2026.1`, Docker documentation (main branch) and Moby v28.5.2, containers-common (main branch); the restart
  loop, `RestartCount` and the Docker debug reload checked on a lab (*Lab observations*), with systemd 255 and Docker,
  not Podman. The start limit was not reached there, so `Start request repeated too quickly.` and `Failed to start`
  were not observed.
- Older systemd releases may lack `--list-invocations`, and the wording of some messages differs between versions.
- The unit names of package installs vary by distribution; not listed exhaustively.

---

*Provenance: written by agent:ozarland-author, from upstream documentation and source code (systemd, Kolla-Ansible,
ansible-collection-kolla, Docker, Moby, containers-common). Reviewed in its pull request (see the page history).
Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs/systemd-journal-and-container-engine.md&title=Feedback%3A+logs/systemd-journal-and-container-engine.md):
your release, your deployment method, and what happened.
