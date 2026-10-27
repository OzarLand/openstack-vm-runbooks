---
title: Reading the time synchronisation logs (chrony, systemd-timesyncd), and how clock skew shows in OpenStack logs
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [chrony, systemd, nova, neutron, keystone]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: any
  storage: any
  os: any linux with systemd
risk: read-only
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-27, note: "systemd-timesyncd (systemd 255) only: NTP blocked (no journal line, Packet count 0, synchronized flag stayed yes), clock set back 120 s and restored (restored-from-recorded-timestamp and initial synchronization lines); not observable on one host: the Neutron clock line, Nova flapping; not run: chrony" }
upstream_refs:
  - https://gitlab.com/chrony/chrony/-/blob/4.9/doc/chronyc.adoc
  - https://gitlab.com/chrony/chrony/-/blob/4.9/doc/chrony.conf.adoc
  - https://gitlab.com/chrony/chrony/-/blob/4.9/reference.c
  - https://gitlab.com/chrony/chrony/-/blob/4.9/sources.c
  - https://gitlab.com/chrony/chrony/-/blob/4.9/sched.c
  - https://gitlab.com/chrony/chrony/-/blob/4.9/local.c
  - https://gitlab.com/chrony/chrony/-/blob/4.9/conf.c
  - https://github.com/systemd/systemd/blob/v259.10/src/timesync/timesyncd-manager.c
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/prechecks/tasks/timesync_checks.yml
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/servicegroup/drivers/db.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/db/main/api.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/conf/service.py
  - https://github.com/openstack/neutron/blob/stable/2026.1/neutron/db/agents_db.py
  - https://github.com/openstack/neutron/blob/stable/2026.1/neutron/conf/agent/database/agents_db.py
  - https://github.com/openstack/keystone/blob/stable/2026.1/keystone/token/provider.py
  - https://github.com/openstack/keystonemiddleware/blob/stable/2026.1/keystonemiddleware/auth_token/__init__.py
  - https://github.com/openstack/neutron/blob/stable/2026.1/neutron/plugins/ml2/drivers/ovn/agent/neutron_agent.py
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Reading the time synchronisation logs (chrony, systemd-timesyncd), and how clock skew shows in OpenStack logs

OpenStack services compare timestamps written by different hosts: service heartbeats, agent reports, token expiry. When
one host's clock is off, some of these comparisons fail, and the logs of that host no longer line up with the others.
This page shows how to read the time service's own logs, and which OpenStack log lines a clock offset produces.

For a compute whose clock drifted, with the fix, see the runbook
[the logs of one compute do not line up](../diagnostics/compute-clock-skew.md).

All commands on this page are read-only.

## Which time service runs

**Kolla-Ansible:** time synchronisation is a host service, not a container. Kolla-Ansible's prechecks look for a
running host NTP daemon (chrony, ntpd or systemd-timesyncd) and fail if `timedatectl` sees the clock as not
synchronised
([`timesync_checks.yml`](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/prechecks/tasks/timesync_checks.yml)).

**Any:**

```bash
# On each host
timedatectl                                   # "System clock synchronized: yes", "NTP service: active"
systemctl is-active chronyd chrony systemd-timesyncd ntp ntpd 2>/dev/null
```

The chrony unit is `chronyd` or `chrony` depending on the distribution.

## chrony

### Its state, with chronyc

From [`chronyc(1)`](https://gitlab.com/chrony/chrony/-/blob/4.9/doc/chronyc.adoc):

```bash
chronyc tracking       # current offset and the selected source
chronyc sources -v     # every source, with its selection state
chronyc sourcestats    # offset and drift estimates per source
chronyc activity       # how many sources are online, offline
```

- In `tracking`, **System time** is the current offset between chrony's NTP clock and the system clock. chrony
  normally corrects it by slewing (speeding up or slowing down the clock), so a large offset can take a very long time
  to disappear; **Last offset** is the estimated offset at the last update. **Leap status** `Not synchronised` means
  chrony is not synchronised.
- In `sources`, the second character is the selection state: `*` the selected source, `+` other sources combined with
  it, `-` selectable but not combined, `x` a falseticker (its time disagrees with the majority), `~` too variable,
  `?` not selectable (for example unreachable).

### Its log lines

chrony logs to syslog, so to the journal (`journalctl -u chronyd` or `-u chrony`). Format strings from chrony 4.9,
not captured lines; `##` and `%s` are replaced by the source name or address:

| Message (format) | Meaning | Source |
|---|---|---|
| `Selected source ##` | chrony now synchronises to this source | `sources.c` |
| `Can't synchronise: no selectable sources` (also `no sources`, `no majority`) | chrony is not synchronising any more: the clock now drifts freely | `sources.c` |
| `Detected falseticker ##` | This source disagrees with the others and is ignored | `sources.c` |
| `Root distance of ## exceeds maxdistance of <n> seconds`, `Jitter of ## exceeds maxjitter of <n> seconds`, `Stratum of ## above maxstratum of <n>`, `Stratum of ## below minstratum of <n>` | This source is rejected | `sources.c` |
| `System clock wrong by <offset> seconds` | An offset larger than `logchange` (1 second by default) was detected | `reference.c`, `conf.c` |
| `System clock wrong by <offset> seconds (step)` or `(ignored)` | With `chronyd -q`/`-Q`, the offset found once | `reference.c` |
| `System clock was stepped by <offset> seconds` | chrony stepped the clock (allowed by `makestep`): every log on this host jumps by that much | `reference.c`, `local.c` |
| `Forward time jump detected!`, `Backward time jump detected!` | chrony saw the clock jump, for example after a manual `date -s` or a VM pause | `sched.c` |
| `No suitable source for synchronisation` | No usable source when chrony needed one | `reference.c` |

`makestep` lets chrony step the clock instead of slewing it, usually only in the first updates after start;
`maxchange` makes it ignore, then exit on, offsets that are too large; both are described in
[`chrony.conf(5)`](https://gitlab.com/chrony/chrony/-/blob/4.9/doc/chrony.conf.adoc).

```bash
journalctl -u chronyd -u chrony --since "-2d" | grep -E "Selected source|Can't synchronise|falseticker|wrong by|stepped|time jump"
```

## systemd-timesyncd

From [`timesyncd-manager.c`](https://github.com/systemd/systemd/blob/v259.10/src/timesync/timesyncd-manager.c),
systemd v259 (format strings):

| Message (format) | Meaning |
|---|---|
| `Contacted time server <address> (<name>).` | First reply from that server |
| `Initial clock synchronization to <date>.` | The clock was set at the first synchronisation; logs of this host jump at that moment |
| `Timed out waiting for reply from <address> (<name>).` | The server did not answer |
| `Server has too large root distance. Disconnecting.` | The server is rejected |
| `Failed to find suitable address for host <name>.` | Name resolution or address family problem |
| `System clock time unset or jumped backwards, restored from recorded timestamp: <date>` | At start, the clock was behind the time timesyncd recorded when it last ran: it was moved forward to that time (seen on a lab with systemd 255) |

On a lab with systemd 255 (Ubuntu 24.04), timesyncd logged **nothing** while its only server was unreachable for 5.5
minutes (NTP blocked, unit restarted): no `Timed out waiting for reply` line; `timedatectl timesync-status` showed
`Packet count: 0` and a growing poll interval, and `timedatectl` still said `System clock synchronized: yes`. Use
`timesync-status` rather than the journal or `timedatectl`'s flag to see a recent loss with timesyncd.

```bash
journalctl -u systemd-timesyncd --since "-2d"
timedatectl timesync-status
```

## How clock skew shows in OpenStack logs

### Nova: services shown down, or up, wrongly

With the default database service group driver, each service's heartbeat stores `last_seen_up = utcnow()` **with the
clock of the process that writes it to the database** (`service_update` in
[`nova/db/main/api.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/db/main/api.py)). For
`nova-compute`, that is `nova-conductor` on a controller; for the controller services, it is the service itself. The
API decides that a service is up when `abs(now - last_seen_up)` is at most `[DEFAULT] service_down_time` (60 s by
default),
with the clock of the host that answers the API call (`is_up` in
[`servicegroup/drivers/db.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/servicegroup/drivers/db.py)).

So a compute's own clock does not matter here, but **an offset of more than about a minute between controllers** makes
services flap between `up` and `down` in `openstack compute service list`, depending on which controller answers.
At debug level, Nova logs:

```text
Seems service <binary> on host <host> is down. Last heartbeat was <time>. Elapsed time is <seconds>
```

A negative elapsed time means the heartbeat is in the future for the host that checks: a clock is behind.

### Neutron: an ERROR at agent start when the agent's clock differs

When an agent starts, it sends its own time with its first report. If it differs from the server's clock by more than
`agent_down_time` (75 s by default), `neutron-server` logs at ERROR (`_check_clock_sync_on_agent_start` in
[`agents_db.py`](https://github.com/openstack/neutron/blob/stable/2026.1/neutron/db/agents_db.py)):

```text
Message received from the host: <host> during the registration of <agent type> has a timestamp: <agent time>. This differs from the current server timestamp: <server time> by <n> seconds, which is more than the threshold agent down time: <n>.
```

This is only a log line: **the agent still registers and reports normally**. Its `heartbeat_timestamp` is set with
the server's clock (`current_time` in `create_or_update_agent`, same file), so the offset does not make the agent show
down either. Do not restart or chase the agent: fix the host's clock. It is the one OpenStack line that names a clock
offset directly, with both times and the difference.

With ML2/OVN this line never appears: the agents that Neutron shows come from the OVN Southbound chassis records
([`ovn/agent/neutron_agent.py`](https://github.com/openstack/neutron/blob/stable/2026.1/neutron/plugins/ml2/drivers/ovn/agent/neutron_agent.py)),
not from agent reports over RPC.

```bash
# On the controllers (Kolla-Ansible path)
grep "This differs from the current server timestamp" /var/log/kolla/neutron/neutron-server.log | tail
```

### Keystone: tokens rejected early, or accepted late

A token's `expires_at` is set by the Keystone node that issues it. The node that validates it compares `expires_at`
with **its own** clock (`_is_valid_token` in
[`keystone/token/provider.py`](https://github.com/openstack/keystone/blob/stable/2026.1/keystone/token/provider.py)),
and an expired token gives `Failed to validate token`. A Keystone node whose clock is ahead rejects tokens before they
expire; the services then return 401 for tokens that look valid.

The API services check expiry too: keystonemiddleware's `auth_token` rejects a token that `will_expire_soon` with a
stale duration of 0, against the API node's own clock, with `Token authorization failed` (`_validate_token` in
[`auth_token/__init__.py`](https://github.com/openstack/keystonemiddleware/blob/stable/2026.1/keystonemiddleware/auth_token/__init__.py)).
So an API node whose clock is ahead returns 401 even when all the Keystone nodes agree.

Runbook: [401 for tokens that should be valid, Fernet keys or a clock](../recovery/keystone-401-for-every-request-fernet-keys-or-clock.md).

### The logs themselves

Each host stamps its logs with its own clock. Before building a timeline across hosts, measure the offset of each
host (`chronyc tracking`, or the same request id on two hosts, as in the
[clock skew runbook](../diagnostics/compute-clock-skew.md)) and correct for it. A
`System clock was stepped by <offset> seconds` or `Initial clock synchronization` line marks the moment a host's
timestamps jumped.

## A short reading order

1. `timedatectl` on every host: which hosts are not synchronised? The flag can stay `yes` for a while after the
   source is lost; with timesyncd, `timedatectl timesync-status` (`Packet count`, `Offset`) tells more.
2. On those hosts, the time service's journal: when did it stop synchronising (`Can't synchronise`, `Timed out
   waiting for reply`), and was the clock stepped since?
3. On the controllers: `neutron-server.log` for `This differs from the current server timestamp`, and the compute
   service list for flapping states.
4. Correct each host's timestamps by its offset before comparing logs.

## Lab observations

Kolla-Ansible 22.2.0 (2026.1) all-in-one with systemd-timesyncd (systemd 255), one NTP server, on 2026-10-27:

- NTP blocked 5.5 min: no timesyncd line, `Packet count: 0`, `System clock synchronized: yes` (see the timesyncd
  section).
- Time service stopped, clock set back 120 s, Neutron OVS agent restarted with `systemctl restart
  kolla-neutron_openvswitch_agent-container.service`, time service started again:

  ```text
  2026-10-27T13:51:10+00:00 compute-1 systemd-journald[361]: Time jumped backwards, rotating.
  2026-10-27T13:53:07+00:00 compute-1 systemd-timesyncd[<pid>]: System clock time unset or jumped backwards, restored from recorded timestamp: Tue 2026-10-27 13:53:07 UTC
  2026-10-27T13:54:42+00:00 compute-1 systemd-timesyncd[<pid>]: Contacted time server 192.0.2.123:123 (192.0.2.123).
  2026-10-27T13:54:42+00:00 compute-1 systemd-timesyncd[<pid>]: Initial clock synchronization to Tue 2026-10-27 13:54:42.992145 UTC.
  ```

- No `This differs from the current server timestamp` line: on an all-in-one the agent and `neutron-server` read the
  same clock. Every Nova service and Neutron agent stayed `up`.
- Not run: chrony (not installed), a Neutron agent on another host than the server, a controller clock ahead.

## Validity and limits

- Written from upstream source and documentation: chrony 4.9, systemd v259.10, Kolla-Ansible, Nova, Neutron and
  Keystone `stable/2026.1`. The timesyncd lines were checked on an all-in-one lab with systemd 255 (*Lab
  observations*); the chrony lines, the Neutron clock line and Nova's flapping were not observed (one host, no
  chrony).
- ntpd is not covered.
- The effects of clock skew on Galera, RabbitMQ and TLS certificate checks are not covered here.
- The reading of a negative elapsed time in Nova's debug line follows from `is_up`; not observed.

---

*Provenance: written by agent:ozarland-author, from upstream documentation and source code (chrony, systemd,
Kolla-Ansible, Nova, Neutron, Keystone). Reviewed in its pull request (see the page history). Checked by a human
operator: not yet.*
