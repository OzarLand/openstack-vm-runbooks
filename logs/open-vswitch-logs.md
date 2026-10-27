---
title: Reading the Open vSwitch logs (ovs-vswitchd, ovsdb-server) and changing their levels with ovs-appctl
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [ovs, neutron]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: ml2/ovs, or ml2/ovn (same Open vSwitch daemons on the computes)
  storage: any
  os: any
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://github.com/openvswitch/ovs/blob/v3.7.1/Documentation/ref/ovs-appctl.8.rst
  - https://github.com/openvswitch/ovs/blob/v3.7.1/lib/vlog.man
  - https://github.com/openvswitch/ovs/blob/v3.7.1/include/openvswitch/vlog.h
  - https://github.com/openvswitch/ovs/blob/v3.7.1/lib/vlog.c
  - https://github.com/openvswitch/ovs/blob/v3.7.1/lib/timeval.c
  - https://github.com/openvswitch/ovs/blob/v3.7.1/lib/ovs-rcu.c
  - https://github.com/openvswitch/ovs/blob/v3.7.1/lib/reconnect.c
  - https://github.com/openvswitch/ovs/blob/v3.7.1/lib/rconn.c
  - https://github.com/openvswitch/ovs/blob/v3.7.1/lib/jsonrpc.c
  - https://github.com/openvswitch/ovs/blob/v3.7.1/lib/memory.c
  - https://github.com/openvswitch/ovs/blob/v3.7.1/vswitchd/bridge.c
  - https://github.com/openvswitch/ovs/blob/v3.7.1/ofproto/connmgr.c
  - https://github.com/openvswitch/ovs/blob/v3.7.1/lib/dpif.c
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/openvswitch/templates/openvswitch-vswitchd.json.j2
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/openvswitch/templates/openvswitch-db-server.json.j2
  - https://github.com/openstack/neutron/blob/stable/2026.1/neutron/conf/plugins/ml2/drivers/ovs_conf.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/network/neutron.py
  - https://github.com/openvswitch/ovs/blob/v3.7.1/ovsdb/ovsdb-server.c
  - https://github.com/ovn-org/ovn/blob/main/controller/ovn-controller.c
  - https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/cron/templates/cron-logrotate-global.conf.j2
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Reading the Open vSwitch logs (ovs-vswitchd, ovsdb-server) and changing their levels with ovs-appctl

Every compute (and, with ML2/OVS, every network node) runs two Open vSwitch daemons, whatever the Neutron backend:

- `ovsdb-server` holds the switch configuration (bridges, ports, interfaces, controllers) in the `Open_vSwitch`
  database;
- `ovs-vswitchd` reads that database, creates the ports in the kernel datapath and forwards packets, following the
  OpenFlow rules set by the Neutron Open vSwitch agent (ML2/OVS) or by `ovn-controller` (ML2/OVN).

Both log through the same library, `vlog`, with the same line format and the same `ovs-appctl vlog/*` commands. The
logs of `ovn-controller` and of the OVN databases use the same format; what their messages mean is covered on the OVN
page.

## Where the logs are

**Kolla-Ansible:** the containers `openvswitch_vswitchd` and `openvswitch_db` start the daemons with
`-vconsole:emer -vsyslog:err -vfile:info` and a log file on the `kolla_logs` volume
([`openvswitch-vswitchd.json.j2`](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/openvswitch/templates/openvswitch-vswitchd.json.j2),
[`openvswitch-db-server.json.j2`](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/openvswitch/templates/openvswitch-db-server.json.j2)):

```text
/var/log/kolla/openvswitch/ovs-vswitchd.log
/var/log/kolla/openvswitch/ovsdb-server.log
```

So the file gets `info` and above, syslog (the journal) only `err` and above, and `docker logs` almost nothing.

**Any (package install):** without a file name, `--log-file` writes to `<LOGDIR>/<program>.log`, where `LOGDIR` is set
when Open vSwitch is built ([`lib/vlog.man`](https://github.com/openvswitch/ovs/blob/v3.7.1/lib/vlog.man)); distribution
packages commonly use `/var/log/openvswitch/`. The running process shows the real path:

```bash
# On the compute
ps -o args= -C ovs-vswitchd | tr ' ' '\n' | grep -E '^--log-file|^-v'
ps -o args= -C ovsdb-server | tr ' ' '\n' | grep -E '^--log-file|^-v'
```

## Reading one line

The default file pattern, from
[`include/openvswitch/vlog.h`](https://github.com/openvswitch/ovs/blob/v3.7.1/include/openvswitch/vlog.h), is
`%D{%Y-%m-%dT%H:%M:%S.###Z}|%05N|%c%T|%p|%m`, which gives this shape (format, not a captured line):

```text
<YYYY-MM-DDTHH:MM:SS.mmmZ>|<serial>|<module>[(<thread>)]|<LEVEL>|<message>
```

| Field | Meaning ([`ovs-appctl(8)`](https://github.com/openvswitch/ovs/blob/v3.7.1/Documentation/ref/ovs-appctl.8.rst)) |
|---|---|
| Timestamp | **UTC** (`Z`), whatever the host's time zone |
| `<serial>` | Message number within this run of the daemon, from `00001`. A jump back to `00001` means the daemon restarted |
| `<module>` | The `vlog` module that logged the message (`bridge`, `connmgr`, `rconn`, `timeval`, …); `ovs-appctl vlog/list` lists them |
| `(<thread>)` | Present when the message comes from a thread other than the main one, for example `(handler12)` or `(revalidator3)` |
| `<LEVEL>` | `EMER`, `ERR`, `WARN`, `INFO` or `DBG` |

The levels, as `ovs-appctl(8)` defines them: `emer` (a major failure forced the process to abort), `err` (a high-level
operation or subsystem failed; attention is warranted), `warn` (a low-level operation failed, higher levels may
recover), `info` (useful in retrospect), `dbg` (too voluminous for normal use, never logged by default).

The serial number is the quickest way to spot a restart:

```bash
# On the compute (Kolla-Ansible path; adapt for a package install)
grep -n '|00001|' /var/log/kolla/openvswitch/ovs-vswitchd.log | tail -5
grep -E '\|(EMER|ERR|WARN)\|' /var/log/kolla/openvswitch/ovs-vswitchd.log | tail -30
```

## Messages worth knowing

All the messages below are format strings from Open vSwitch v3.7.1, not captured lines.

### Ports and interfaces (module `bridge`)

| Message (format) | Meaning |
|---|---|
| `bridge <bridge>: added interface <name> on port <n>` | A port from the database is now in the datapath. For an instance, `<name>` is its tap device (`tap<port-id prefix>`) on `br-int` |
| `bridge <bridge>: deleted interface <name> on port <n>` | The port was removed |
| `could not open network device <name> (<error>)` | The database has an interface that `ovs-vswitchd` cannot open. `No such device` for a `tap…` name usually means the instance or its tap is gone while the port is still in the database |
| `could not add network device <name> to ofproto (<error>)` | The device exists but could not be added to the bridge |
| `bridge <bridge>: using datapath ID <dpid>` | The bridge was (re)configured; seen at start-up |

To match a tap device to a Neutron port: Nova names the tap `tap` followed by the port id, cut to 14 characters, so
`tap` and the first 11 characters of the port id (`NIC_NAME_LEN` in
[`nova/network/neutron.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/network/neutron.py) and
`model.py`). The database keeps the full id in `external_ids:iface-id`:

```bash
# On the compute. Kolla-Ansible: prefix with docker exec openvswitch_vswitchd
ovs-vsctl --columns=name,external_ids find Interface name=tap<port-id-prefix>
```

### Controllers and database connections (modules `rconn`, `reconnect`, `connmgr`, `jsonrpc`)

`ovs-vswitchd` is an OpenFlow switch: its controller is the Neutron Open vSwitch agent (ML2/OVS, which listens by
default on `127.0.0.1:6633`, options `of_listen_address` and `of_listen_port` in
[`ovs_conf.py`](https://github.com/openstack/neutron/blob/stable/2026.1/neutron/conf/plugins/ml2/drivers/ovs_conf.py)),
or `ovn-controller` (ML2/OVN). A controller connection is named `<bridge><-><target>`, for example
`br-int<->tcp:127.0.0.1:6633` ([`connmgr.c`](https://github.com/openvswitch/ovs/blob/v3.7.1/ofproto/connmgr.c)).

With ML2/OVN, `ovn-controller` is not in the bridge's Controller table: it connects as a client to the bridge's
management socket, `unix:<rundir>/br-int.mgmt`
([`ovn-controller.c`](https://github.com/ovn-org/ovn/blob/main/controller/ovn-controller.c)). Failures on that link
are logged by `ovn-controller`, in `ovn-controller.log`, not as `br-int<->…` lines in `ovs-vswitchd.log`.

| Message (format) | Meaning | Source |
|---|---|---|
| `<name>: connecting...` then `<name>: connected` | (Re)connection to a controller or to the database. For OpenFlow, `rconn.c` logs `connected` at INFO only for reliable connections (such as a configured controller), and at DBG otherwise | `rconn.c`, `reconnect.c` |
| `<name>: connection failed (<error>)`, `<name>: connection timed out` | An OpenFlow controller is not answering. `Connection refused` on `127.0.0.1:6633` means the Neutron agent is not listening (stopped, restarting) | `rconn.c` |
| `<name>: connection attempt failed (<error>)`, `<name>: connection attempt timed out` | The same for a database (JSON-RPC) connection, such as a client of `ovsdb-server` | `reconnect.c` |
| `<name>: connection closed by peer`, `<name>: connection dropped (<error>)` | The other side went away | `reconnect.c` |
| `<name>: no response to inactivity probe after <n> seconds, disconnecting` | The other side is alive enough to keep the socket open but did not answer for `<n>` seconds: an overloaded or blocked agent, `ovsdb-server` or `ovs-vswitchd` | `reconnect.c`, `rconn.c` |
| `<name>: waiting <n> seconds before reconnect` | Back-off between attempts; the wait doubles up to a maximum. Logged by `rconn.c` for OpenFlow connections (whole seconds) and by `reconnect.c` for database connections | `rconn.c`, `reconnect.c` |
| `<name>: receive error: <error>`, `<name>: send error: <error>` | A JSON-RPC (database) connection failed; `<name>` such as `unix#<n>` is a client of the unix socket | `jsonrpc.c` |

A controller connection that drops and comes back while the Neutron agent restarts is expected; the agent's restart
does not cut the instances
([restarting the OVS agent](../networking/restarting-the-ovs-agent-does-not-cut-instances.md)).

### The daemon is struggling (modules `timeval`, `ovs_rcu`, `memory`, `vlog`)

| Message (format) | Meaning | Source |
|---|---|---|
| `Unreasonably long <n>ms poll interval (<u>ms user, <s>ms system)`, sometimes followed by `faults: …`, `disk: …`, `context switches: …` | The main loop of the daemon did not come back for at least one second. `user` high: the daemon itself was busy (huge flow table, many ports). `system` high, or `faults` with major faults: the host is short of memory or CPU. Correlate with the kernel log and the host load | `lib/timeval.c` |
| `blocked <n> ms waiting for <thread> to quiesce` | A thread is stuck; with repeated lines, the daemon is close to hung | `lib/ovs-rcu.c` |
| `<n> kB peak resident set size after <t> seconds`, then `peak resident set size grew <p>% in last <t> seconds, from <a> kB to <b> kB` | Memory growth report (the first at start-up, then each growth of 50 % or more) | `lib/memory.c` |
| `Dropped <n> log messages in last <t> seconds (most recently, <s> seconds ago) due to excessive rate` | Rate limiting hid messages: the problem is bigger than the lines around it show | `lib/vlog.c` |
| `<datapath>: failed to add <name> as port: <error>` | The kernel datapath refused a port | `lib/dpif.c` |

When `ovs-vswitchd` stops, its log simply ends; the next start begins again at serial `00001`. See
[ovs-vswitchd down on a compute](../networking/ovs-vswitchd-down-on-compute.md) for what that does to the instances.

### ovsdb-server

`ovsdb-server` logs its version at start-up (`ovsdb-server (Open vSwitch) <version>`), `loading configuration from
'<file>'`, and failures such as `failed to open database '<name>': <error>` or `failed to configure remotes: <error>`
([`ovsdb-server.c`](https://github.com/openvswitch/ovs/blob/v3.7.1/ovsdb/ovsdb-server.c)). Its client connections use
the `jsonrpc` and `reconnect` messages above. `ovs-vswitchd`, `ovs-vsctl`, the Neutron agent and `ovn-controller` are
all clients.

## Changing the log levels with ovs-appctl

`ovs-appctl` talks to a running daemon through its control socket. By default it targets `ovs-vswitchd`; `-t
ovsdb-server` targets the database server. From
[`ovs-appctl(8)`](https://github.com/openvswitch/ovs/blob/v3.7.1/Documentation/ref/ovs-appctl.8.rst):

```bash
# On the compute (read-only). Kolla-Ansible: docker exec openvswitch_vswitchd ovs-appctl ...
#                                         and docker exec openvswitch_db ovs-appctl -t ovsdb-server ...
ovs-appctl vlog/list                 # modules, and the level of each for console, syslog and file
ovs-appctl vlog/list-pattern         # the pattern of each destination
ovs-appctl -t ovsdb-server vlog/list
```

`vlog/set` takes up to one word of each kind: a module (from `vlog/list`), a destination (`syslog`, `console`,
`file`), a level (`off`, `emer`, `err`, `warn`, `info`, `dbg`). Messages at the level or above are kept. Logging to a
file only happens if the daemon was started with `--log-file`.

> ⚠️ **State-changing.** Debug logging can grow the log file very fast on a busy compute (some modules log every
> packet or flow) and costs CPU. On Kolla-Ansible, `/var/log/kolla` is shared by every service of the host, so a
> runaway log can fill the disk for all of them. Instances are not affected. The change lasts until you undo it or the
> daemon restarts. Raise one module at a time, on the destination `file`, and note the level shown by `vlog/list`
> before you change it. Many debug messages are rate-limited; `ovs-appctl vlog/disable-rate-limit <module>` lifts that
> and adds even more volume: use it only briefly, and undo it with `vlog/enable-rate-limit <module>`.

```bash
# On the compute. Kolla-Ansible: docker exec openvswitch_vswitchd ovs-appctl ...
ovs-appctl vlog/list | grep -E 'console|bridge|rconn'          # note the current levels (columns: console, syslog, file)
ovs-appctl vlog/set bridge:file:dbg                            # one module, file only
ovs-appctl vlog/set rconn:file:dbg
```

Undo by setting back the level you noted, `info` with Kolla-Ansible's `-vfile:info`:

```bash
ovs-appctl vlog/set bridge:file:info
ovs-appctl vlog/set rconn:file:info
```

`ovs-appctl vlog/set` without any argument sets **every** module and destination to `dbg`: avoid it on a production
compute.

### Rotation

`ovs-appctl vlog/reopen` makes the daemon close and reopen its log file, which is what a log rotation needs after
moving the file. Neither it nor `vlog/close` has an effect if the daemon was started without `--log-file`.
Kolla-Ansible's logrotate configuration uses `copytruncate`
([`cron-logrotate-global.conf.j2`](https://github.com/openstack/kolla-ansible/blob/stable/2026.1/ansible/roles/cron/templates/cron-logrotate-global.conf.j2)),
so `vlog/reopen` is rarely needed there.

Do not use `vlog/close` on a production host: the daemon then stops writing its log file until `vlog/reopen` or a
restart, and the next incident leaves no trace.

## A short reading order

1. Network lost for every instance of one compute: is `ovs-vswitchd` running, and did its log restart at `00001`?
2. One instance without network: `added interface tap…` for its port at boot, or `could not open network device`?
3. Flows missing or stale: with ML2/OVS, `connection failed` or `no response to inactivity probe` towards the
   Neutron agent in `ovs-vswitchd.log`, then the agent's own log. With ML2/OVN, read `ovn-controller.log`: the
   OpenFlow link to `br-int.mgmt` is logged there.
4. Slowness, drops under load: `Unreasonably long … poll interval` and `blocked … waiting … to quiesce`, then the
   kernel log and host load for the same UTC minute.

Related runbooks: [ovs-vswitchd down on a compute](../networking/ovs-vswitchd-down-on-compute.md),
[tunnel MTU mismatch](../networking/tunnel-mtu-mismatch-large-packets-dropped.md),
[restarting the OVS agent](../networking/restarting-the-ovs-agent-does-not-cut-instances.md),
[after a compute reboot](../recovery/after-a-compute-reboot-instances-shutoff-or-without-network.md).

## Validity and limits

- Written from upstream source and documentation only: Open vSwitch v3.7.1, Kolla-Ansible and Neutron
  `stable/2026.1`. Every message above is a format string from that source, not a captured line. Older Open vSwitch
  releases may word some lines differently (`ovs-vswitchd --version` gives yours). The Open vSwitch version on a
  2026.1 compute depends on the base distribution of the images or packages, so it may not be 3.7.1.
- Not run on a lab yet. A capture of a compute with an instance booted and deleted, the Neutron agent restarted, and a
  module raised to `dbg` and back, is requested to replace the formats with redacted real lines.
- DPDK (`ovs-vswitchd` with a userspace datapath) adds its own modules and messages; not covered.
- The meaning given for `Unreasonably long … poll interval` with high `system` time is an interpretation of what the
  numbers measure (`getrusage` of the thread), not an upstream statement.

---

*Provenance: written by agent:ozarland-author, from upstream documentation and source code (Open vSwitch,
Kolla-Ansible, Neutron). Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*
