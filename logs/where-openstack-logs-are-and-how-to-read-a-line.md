---
title: Where the OpenStack logs are, and how to read one log line
kind: diagnostic
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, neutron, placement, keystone, glance, cinder, rabbitmq, mariadb, haproxy, ovs, ovn, libvirt]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: any
  storage: any
  os: any
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-27, note: "Kolla part: log directories and files, read commands with sudo, nova_compute debug on/off with systemctl restart (not docker restart), GMR via SIGUSR2 into docker logs and the unit journal, real lines of oslo.log, uWSGI, OVS, RabbitMQ, libvirtd, MariaDB; not run: package-install commands, journald fields, HAProxy and OVN logs, a GMR under uWSGI" }
upstream_refs:
  - https://opendev.org/openstack/oslo.log/src/branch/stable/2026.1/oslo_log/_options.py
  - https://opendev.org/openstack/oslo.log/src/branch/stable/2026.1/oslo_log/formatters.py
  - https://opendev.org/openstack/oslo.log/src/branch/stable/2026.1/oslo_log/handlers.py
  - https://opendev.org/openstack/oslo.context/src/branch/stable/2026.1/oslo_context/context.py
  - https://opendev.org/openstack/oslo.middleware/src/branch/stable/2026.1/oslo_middleware/request_id.py
  - https://docs.openstack.org/kolla-ansible/latest/user/troubleshooting.html
  - https://opendev.org/openstack/kolla-ansible/src/branch/stable/2026.1/ansible/roles/fluentd/templates/conf/input/00-global.conf.j2
  - https://docs.openstack.org/nova/latest/admin/configuration/logs.html
  - https://docs.openstack.org/nova/latest/reference/gmr.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Where the OpenStack logs are, and how to read one log line

Every investigation starts with two questions: where does this service write, and what does each field of its line
mean. The answers depend on how the platform was deployed, and much less on the service, because almost every
OpenStack service logs through the same library, `oslo.log`, with the same line format.

This page covers:

- where the logs are on a Kolla-Ansible deployment and on a package (distribution) install;
- the default `oslo.log` line, field by field, including the request ids and the instance prefix;
- the log levels, and how to turn debug on for one service and back off;
- the logs that do not use `oslo.log` (RabbitMQ, MariaDB, HAProxy, Open vSwitch, OVN, libvirt), with their own
  formats.

How to follow one request across services is on its own page:
[what a VM creation traverses, and tracing by request id](../diagnostics/what-a-vm-creation-traverses-and-tracing-by-request-id.md).

## Where the logs are

### Kolla-Ansible

Every container writes into one Docker (or Podman) volume, `kolla_logs`, mounted at `/var/log/kolla` in the
containers. On the host, `/var/log/kolla` points to that volume, one directory per service
([Kolla-Ansible troubleshooting guide](https://docs.openstack.org/kolla-ansible/latest/user/troubleshooting.html)):

```bash
# On any node: where the log volume really is (sudo: /var/lib/docker is readable by root only)
sudo readlink -f /var/log/kolla
# Kolla-Ansible's guide shows /var/lib/docker/volumes/kolla_logs/_data

sudo ls /var/log/kolla/
```

The file names come from the `log_file` and `log_dir` options that Kolla-Ansible 2026.1 writes into each service's
configuration. With `log_dir` alone, `oslo.log` names the file after the program: `<log_dir>/<program>.log`
([`_get_log_file_path`](https://opendev.org/openstack/oslo.log/src/branch/stable/2026.1/oslo_log/log.py)).

| Service | Container (default name) | Log file | Where it runs |
|---|---|---|---|
| nova-api | `nova_api` | `/var/log/kolla/nova/nova-api.log`, requests in `nova-api-uwsgi.log` | controllers |
| nova-scheduler | `nova_scheduler` | `/var/log/kolla/nova/nova-scheduler.log` | controllers |
| nova-conductor | `nova_conductor` | `/var/log/kolla/nova/nova-conductor.log` (cell conductor) | controllers |
| nova super conductor | `nova_super_conductor` | `/var/log/kolla/nova/nova-super-conductor.log` (only with `enable_cells: true`) | controllers |
| nova-compute | `nova_compute` | `/var/log/kolla/nova/nova-compute.log` | computes |
| libvirt | `nova_libvirt` | `/var/log/kolla/libvirt/libvirtd.log`; each instance's QEMU log is libvirt's `/var/log/libvirt/qemu/<instance-name>.log` inside the container (not under `kolla_logs`, see the limits) | computes |
| placement-api | `placement_api` | `/var/log/kolla/placement/placement-api.log`, requests in `placement-api-uwsgi.log` | controllers |
| neutron-server (API) | `neutron_server` | `/var/log/kolla/neutron/neutron-server.log`, requests in `neutron-server-uwsgi.log` | controllers |
| Neutron RPC server, periodic workers | `neutron_rpc_server`, `neutron_periodic_worker` | `/var/log/kolla/neutron/neutron-rpc-server.log`, `neutron-periodic-workers.log` (named after the program) | controllers |
| Neutron agents | `neutron_openvswitch_agent`, `neutron_l3_agent`, `neutron_dhcp_agent`, `neutron_metadata_agent`, `neutron_ovn_metadata_agent` | `/var/log/kolla/neutron/<agent program>.log` | network nodes, computes |
| keystone | `keystone` | `/var/log/kolla/keystone/keystone.log` | controllers |
| glance-api | `glance_api` | `/var/log/kolla/glance/glance-api.log` | controllers |
| cinder | `cinder_api`, `cinder_scheduler`, `cinder_volume` | `/var/log/kolla/cinder/cinder-api.log`, `cinder-scheduler.log`, `cinder-volume.log` | controllers, storage nodes |
| RabbitMQ | `rabbitmq` | `/var/log/kolla/rabbitmq/rabbit@<hostname>.log` | controllers |
| MariaDB / Galera | `mariadb` | `/var/log/kolla/mariadb/mariadb.log` | controllers |
| HAProxy | `haproxy` | under `/var/log/kolla/haproxy/` (sent through syslog to fluentd, file `haproxy_latest`) | load balancer nodes |
| ProxySQL (database proxy, default with MariaDB) | `proxysql` | `/var/log/kolla/proxysql/proxysql.log` (`errorlog` in `proxysql.yaml.j2`) | load balancer nodes |
| Open vSwitch | `openvswitch_vswitchd`, `openvswitch_db` | `/var/log/kolla/openvswitch/ovs-vswitchd.log`, `ovsdb-server.log` | computes, network nodes |
| OVN | `ovn_controller`, `ovn_northd`, `ovn_nb_db`, `ovn_sb_db` | `/var/log/kolla/openvswitch/ovn-controller.log`, `ovn-northd.log`, `ovn-nb-db.log`, `ovn-sb-db.log` | computes, controllers |

Sources for the table: the 2026.1 templates `nova.conf.j2` (`nova` and `nova-cell` roles), `neutron.conf.j2`,
`placement.conf.j2`, `keystone.conf.j2`, `glance-api.conf.j2`, `cinder.conf.j2`, the `service-uwsgi-config` role
(`logto2 = /var/log/kolla/<role>/<service>-uwsgi.log`), and fluentd's input definitions, which list the files it
reads (`input/02-mariadb.conf.j2`, `03-rabbitmq.conf.j2`, `05-libvirt.conf.j2`, `10-openvswitch.conf.j2`) and
where it writes HAProxy's syslog stream (`syslog_facilities` in the `fluentd` role defaults). Directory and file
names can differ when a service is disabled, renamed or overridden; `ls /var/log/kolla/<service>/` is the truth on
your platform.

Things that surprise people on Kolla-Ansible:

- **`docker logs` is mostly empty.** Most containers do not log to standard output, so `docker logs <container>`
  shows only the start-up of the container (the configuration copy by `kolla_set_configs`) and crashes before
  `oslo.log` is set up. Read the files instead; `docker logs` is useful when a container restarts in a loop.
- **The API services run under uWSGI.** In 2026.1 the API services (Nova, Neutron, Placement, Glance, Cinder,
  Keystone and others) write two files: the application log (`<service>.log`, `oslo.log` format) and the uWSGI log
  (`<service>-uwsgi.log`), which carries one line per HTTP request in uWSGI's own format, starting with
  `[pid: <pid>|app: <n>|req: <n>/<n>] <client address> ...`.
- **fluentd reads the files, it does not replace them.** The `fluentd` container (enabled by default,
  `enable_fluentd: true`) tails the files above. It reads the systemd journal only with central logging
  (`enable_fluentd_systemd` is `enable_fluentd and enable_central_logging` in `group_vars/all/fluentd.yml`), and it
  sends logs to OpenSearch only when `enable_central_logging` is true (default `false`). Without central logging, the
  files on each node are all there is.
- **The host's own logs are outside `/var/log/kolla`.** The kernel, the container engine, chrony and systemd units
  log to the journal of the host (`journalctl`), as on any Linux host.

**Kolla-Ansible:** read a service's log on the node where it runs:

```bash
# sudo: the files are readable by root only, and so is the Docker socket
sudo tail -f /var/log/kolla/nova/nova-compute.log
sudo grep -nE ' [0-9]+ (ERROR|CRITICAL) ' /var/log/kolla/nova/nova-compute.log | tail -20
# the container engine's view of one container: start-up, crashes, restarts
sudo docker ps -a --filter name=nova_compute
sudo docker logs --tail 50 nova_compute     # podman logs on a Podman deployment
```

### Package installs (distribution packages)

*(Not run on the lab: the lab was Kolla-Ansible.)*

Nova's documentation gives `/var/log/nova/` as the directory of its log files, one per service, with service
names that differ by distribution
([Compute log files](https://docs.openstack.org/nova/latest/admin/configuration/logs.html)):

| Log file | Service on RHEL, CentOS, Fedora | Service on Ubuntu, Debian |
|---|---|---|
| `nova-api.log` | `openstack-nova-api` | `nova-api` |
| `nova-compute.log` | `openstack-nova-compute` | `nova-compute` |
| `nova-conductor.log` | `openstack-nova-conductor` | `nova-conductor` |
| `nova-scheduler.log` | `openstack-nova-scheduler` | `nova-scheduler` |

The other projects follow the same pattern (`/var/log/neutron/`, `/var/log/cinder/`, and so on), but where a
service writes depends on its packaging, not on OpenStack. `oslo.log` adds one handler for each output that is
configured, and they add up: a service can write to a file and to the journal at the same time
([`_setup_logging_from_conf`](https://opendev.org/openstack/oslo.log/src/branch/stable/2026.1/oslo_log/log.py)):

- `log_file` and/or `log_dir` set (in the configuration file, or as `--log-file` on the command line of the unit):
  a file;
- `use_stderr = true`: standard error;
- `use_journal = true`: the systemd journal, natively, with structured fields;
- `use_syslog = true`: syslog;
- none of the first three: standard output, which systemd captures in the journal.

So on a package install, look in both places:

**Any:**

```bash
# What the unit runs, with which --log-file or --config-file
systemctl cat <unit>                     # e.g. openstack-nova-compute or nova-compute
# What the journal holds for it
journalctl -u <unit> --since "1 hour ago"
# The configured log options
grep -E '^\s*(log_file|log_dir|use_journal|use_syslog|use_stderr|debug)\s*=' /etc/nova/nova.conf
```

When a service logs natively to the journal (`use_journal = true`), each entry carries structured fields set by
`oslo.log`'s journal handler: `SYSLOG_IDENTIFIER` (the program name), `PRIORITY`, `LOGGER_NAME`,
`LOGGER_LEVEL`, `CODE_FILE`, `CODE_LINE`, `CODE_FUNC`, and, when known, `REQUEST_ID`, `PROJECT_ID`,
`PROJECT_NAME`, `USER_ID`, `USER_NAME`, plus `EXCEPTION_INFO` for a traceback
([`OSJournalHandler`](https://opendev.org/openstack/oslo.log/src/branch/stable/2026.1/oslo_log/handlers.py)).
That lets you filter by field instead of by text:

```bash
journalctl REQUEST_ID=<request-id>                        # not run on the lab
journalctl SYSLOG_IDENTIFIER=nova-compute PRIORITY=3      # errors only (3 = err)
```

These fields exist only with `use_journal = true`. A service that writes to standard output, captured by
systemd, has plain text lines in the journal and none of these fields.

## Reading one oslo.log line

### The default format

`oslo.log` uses one of two formats, chosen per line
([`ContextFormatter.format`](https://opendev.org/openstack/oslo.log/src/branch/stable/2026.1/oslo_log/formatters.py)):

- when the line was logged inside a request (it has a request id), `logging_context_format_string`, whose default
  in 2026.1 is

  ```text
  %(asctime)s.%(msecs)03d %(process)d %(levelname)s %(name)s [%(global_request_id)s %(request_id)s %(user_identity)s] %(instance)s%(message)s
  ```

- otherwise (periodic tasks, start-up, background threads), `logging_default_format_string`:

  ```text
  %(asctime)s.%(msecs)03d %(process)d %(levelname)s %(name)s [-] %(instance)s%(message)s
  ```

Both defaults are in
[`oslo_log/_options.py`](https://opendev.org/openstack/oslo.log/src/branch/stable/2026.1/oslo_log/_options.py).
A deployment can change them in the `[DEFAULT]` section of each service, so check that first if your lines do not
look like this. Kolla-Ansible 2026.1 does not override them: its fluentd parser expects exactly this shape (the
`format1` expression in
[`input/00-global.conf.j2`](https://opendev.org/openstack/kolla-ansible/src/branch/stable/2026.1/ansible/roles/fluentd/templates/conf/input/00-global.conf.j2)).

### Field by field

The shape of a line logged inside a request, with placeholders (this is the format above, not a captured line):

```text
<date> <time>.<ms> <pid> <LEVEL> <module> [<global-request-id or None> req-<uuid> <user-id> <project-id> <domain-id> <system-scope> <user-domain-id> <project-domain-id>] [instance: <instance-uuid>] <message>
```

| Field | Format element | What it tells you |
|---|---|---|
| `<date> <time>.<ms>` | `%(asctime)s.%(msecs)03d`, date format `%Y-%m-%d %H:%M:%S` | The **local time of the host** that wrote the line, without a time zone. Comparing hosts assumes their clocks agree ([clock skew](../diagnostics/compute-clock-skew.md)). |
| `<pid>` | `%(process)d` | The process id. An API or a conductor runs several workers: lines of one request stay in one worker; a restart shows as a new pid. |
| `<LEVEL>` | `%(levelname)s` | `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL` (see below). |
| `<module>` | `%(name)s` | The Python logger, usually the module that logged: `nova.compute.manager`, `nova.scheduler.manager`, `oslo_messaging._drivers.impl_rabbit`. It tells you which part of the code speaks, and whether it is the service itself or a library (`oslo_messaging`, `oslo_db`, `keystonemiddleware`). |
| `<global-request-id>` | `%(global_request_id)s` | The id of the request that **started** the chain in another service, when the caller passed one. `None` when the request started here. |
| `req-<uuid>` | `%(request_id)s` | The id of the request **in this service**, generated by `oslo.middleware` as `req-<uuid4>` for each API call and returned to the client in the `x-openstack-request-id` response header. |
| `<user-id> ... <project-domain-id>` | `%(user_identity)s` | Who made the request: `logging_user_identity_format`, default `%(user)s %(project)s %(domain)s %(system_scope)s %(user_domain)s %(project_domain)s`. These are **ids**, not names; a missing value prints `-`. |
| `[instance: <uuid>] ` | `%(instance)s` | Present when the code passed the instance (Nova does so for most compute actions): `instance_format` / `instance_uuid_format`, default `[instance: %(uuid)s] `. Empty otherwise. This is what makes `grep <instance-uuid>` work across Nova's logs. |
| `<message>` | `%(message)s` | The message. When the line carries an exception, `oslo.log` appends `: <error summary>` (the exception's type and text) if the format does not place it elsewhere. |

Three details that matter when you grep:

- **`[-]` means "no request".** Lines without a request id use the default format, with a literal `[-]`. Periodic
  tasks (resource tracker updates, power state sync, agent reports) log this way: they belong to no API call, so
  you will not find them by request id. Find them by time and by module instead.
- **A traceback is many lines, each with a prefix.** When an exception is logged, every line of the traceback is
  printed with `logging_exception_prefix`, default `%(asctime)s.%(msecs)03d %(process)d ERROR %(name)s %(instance)s`.
  Those lines carry **no request id**. To read a traceback that belongs to a request, find the request's `ERROR`
  line, then read the following lines from the same pid and timestamp:

  ```bash
  grep -n "<request-id>" /var/log/kolla/nova/nova-compute.log | grep ERROR     # line numbers
  sed -n '<line>,+40p' /var/log/kolla/nova/nova-compute.log                  # the traceback after it
  ```

- **DEBUG lines are longer.** At debug level, `oslo.log` appends `logging_debug_format_suffix`, default
  `%(funcName)s %(pathname)s:%(lineno)d`: the function, file and line that logged. Useful to read the code that
  produced the line; noise otherwise.

### Request id and global request id

The two ids answer different questions
([`oslo_middleware/request_id.py`](https://opendev.org/openstack/oslo.middleware/src/branch/stable/2026.1/oslo_middleware/request_id.py),
[`oslo_context/context.py`](https://opendev.org/openstack/oslo.context/src/branch/stable/2026.1/oslo_context/context.py)):

- Each API service **generates its own** `req-<uuid>` for every request it receives, and returns it in the
  `x-openstack-request-id` header.
- When a service calls another one, it sends an `X-Openstack-Request-Id` header with its **global id**: its own
  global request id if it has one, otherwise its request id (`RequestContext.global_id`). The receiving service
  accepts it only if it has the exact `req-<uuid>` shape, and stores it as its `global_request_id`.

So, for one `openstack server create`:

- `nova-api` logs `[None req-A ...]`: the request started in Nova;
- the Neutron calls that Nova makes for this instance appear in `neutron-server.log` as `[req-A req-N ...]`: the
  global id is Nova's, the local one is Neutron's.

To follow one user action across services, grep for the **id from the first service** (`req-A`): it appears as the
request id there and as the global request id in the services it called. Whether a given call carries the global id
depends on the calling code passing its context; check it on your release before you rely on it for a given hop.
The tracing method itself is on the
[tracing page](../diagnostics/what-a-vm-creation-traverses-and-tracing-by-request-id.md).

## Log levels

`oslo.log` uses the Python levels, plus two of its own
([`oslo_log/handlers.py`](https://opendev.org/openstack/oslo.log/src/branch/stable/2026.1/oslo_log/handlers.py),
[`oslo_log/log.py`](https://opendev.org/openstack/oslo.log/src/branch/stable/2026.1/oslo_log/log.py)):

| Level | Value | How to read it |
|---|---|---|
| `CRITICAL` | 50 | The service is about to stop, or cannot start. Read the lines just before it. |
| `ERROR` | 40 | An operation failed. Not always the cause: a failure in a dependency (database, bus, another API) shows as an `ERROR` in every service that used it. Find the first one in time. |
| `WARNING` | 30 | Something unexpected that the service worked around: a retry, a timeout that recovered, a deprecated option. A burst of warnings often precedes an error. |
| `INFO` | 20 | The normal story: requests served, instances spawned, agents reporting. Default level. |
| `DEBUG` | 10 | Each step, payloads, RPC calls. Off by default. |
| `TRACE` | 5 | `oslo.log` defines it; rarely used by the services. |

The default level is `INFO` (`debug = false`). Some noisy libraries are pinned higher whatever the service level,
by `default_log_levels`: for example `amqp=WARN`, `sqlalchemy=WARN`, `keystonemiddleware=WARN`,
`oslo.messaging=INFO`, `urllib3.connectionpool=WARN`. That is why `debug = true` on a service does not show the SQL
it runs or every keystonemiddleware step; to see one of those, change its entry in `default_log_levels` for that
service.

## Turning debug on for one service, then off

Debug on a busy controller writes a lot, quickly, and can fill the disk that holds the logs. Turn it on for **one
service on one host**, for the minutes you need, and turn it off.

**Kolla-Ansible:** each container reads its configuration from `/etc/kolla/<container-config-dir>/` on the host,
copied into the container at start (`config_strategy: COPY_ALWAYS` by default). Kolla-Ansible writes
`debug = {{ <service>_logging_debug }}` in the `[DEFAULT]` section, which is `False` unless
`openstack_logging_debug` or `<service>_logging_debug` is set in `globals.yml`.

```bash
# On the host that runs the service (example: nova-compute on one compute)
sudo grep -n '^debug' /etc/kolla/nova-compute/nova.conf     # expect debug = False; the first match is in [DEFAULT]
sudo df -h "$(sudo readlink -f /var/log/kolla)"             # room for the extra log volume (Docker or Podman)
```

> ⚠️ **State-changing.** Edits the service's configuration on this host and restarts its container. The restart can
> take about a minute (62 s on the lab), because the stop waits up to `docker_graceful_timeout` (60 s by default);
> the service is unavailable meanwhile: requests in flight to it are lost or retried, and for `nova_compute` the
> instances keep running (see [nova-compute down, instances still running](../recovery/nova-compute-down-instances-still-running.md)).
> Do it on one host only. Undo with the commands below, or with the next `kolla-ansible reconfigure`, which rewrites
> the file from its template.

```bash
# first match only: the one in [DEFAULT] (nova.conf also has debug in [glance] and [guestfs])
sudo sed -i '0,/^debug = False/s//debug = True/' /etc/kolla/nova-compute/nova.conf
sudo systemctl restart kolla-nova_compute-container.service
sudo grep -c ' DEBUG ' /var/log/kolla/nova/nova-compute.log # the count grows
```

Back off, as soon as you have what you need:

> ⚠️ **State-changing.** Same restart as above.

```bash
sudo sed -i '0,/^debug = True/s//debug = False/' /etc/kolla/nova-compute/nova.conf
sudo systemctl restart kolla-nova_compute-container.service
```

To keep debug for a whole service across all hosts, set `<service>_logging_debug: "True"` (for example
`nova_logging_debug`) in `globals.yml` and run `kolla-ansible reconfigure` for that service: that restarts the
service everywhere, and needs the same reconfigure to undo.

**Any:** set `debug = true` in the `[DEFAULT]` section of the service's configuration file, restart the unit, and
undo the same way.

> ⚠️ **State-changing.** Restarts the service on this host; same impact as above.

```bash
crudini --set /etc/nova/nova.conf DEFAULT debug true      # or edit the file by hand
systemctl restart <unit>
```

`debug` is declared `mutable` in `oslo.log`, so services that reload mutable options on `SIGHUP` (through
`oslo.service`) can pick it up without a restart. What else a `SIGHUP` does is service-specific (for example,
`nova-scheduler` refreshes its cell cache, per Nova's
[cells documentation](https://docs.openstack.org/nova/latest/admin/cells.html)); this page sticks to the restart,
whose effects are known.

## Guru Meditation report: the state of a live process

When a service hangs without logging anything, its log cannot tell you why. Nova and most OpenStack services can
dump a *Guru Meditation report* (GMR): version, configuration, the stack of every thread and green thread. Sending
`SIGUSR2` to the process writes it to the process's standard error, or to a file in `[oslo_reports] log_dir` when set
([Nova GMR reference](https://docs.openstack.org/nova/latest/reference/gmr.html)). Services run under uWSGI intercept
`SIGUSR2`: for them, configure `[oslo_reports] file_event_handler` and touch that file *(not run on the lab)*.

**Any:**

```bash
sudo kill -USR2 <pid-of-the-service>
```

On Kolla-Ansible, standard error of the service is the container's output, so the report appears in
`docker logs <container>` and in the journal of its unit (`journalctl -u kolla-<container>-container.service`).
On a lab, `sudo kill -USR2` on the `nova-compute` process (its host PID from `sudo docker top nova_compute`) wrote a
report of more than 2000 lines there at once, starting with `====  Guru Meditation  ====`, then `Package`,
`Threads`, ...; the service kept running.

## Logs that do not use oslo.log

The infrastructure around OpenStack has its own formats. Kolla-Ansible's fluentd parsers describe them; each has
its own page in this folder (to come).

| Log | Line starts with | Source of the format |
|---|---|---|
| RabbitMQ | `<date> <time>.<ms>[<digits><+HH:MM>] [<level>] <message>` | fluentd `input/03-rabbitmq.conf.j2` |
| MariaDB / Galera | a date, `YYYY-MM-DD` or `YYMMDD` (the server and its `mysqld_safe` wrapper use different formats) | fluentd `input/02-mariadb.conf.j2` |
| Open vSwitch, OVN | `<date>T<time>.<ms>Z\|<sequence>\|<module>\|<level>\|<message>` (UTC) | fluentd `input/10-openvswitch.conf.j2` |
| libvirtd | `<date> <time>.<ms><offset>: <pid>: <level> : <message>` | fluentd `input/05-libvirt.conf.j2` |
| uWSGI request lines | `[pid: <pid>\|app: <n>\|req: <n>/<n>] <client> (<user>) {<n> vars in <n> bytes} [<date>] ...` | fluentd `input/13-uwsgi.conf.j2` |

Two of these are in **UTC** (Open vSwitch and OVN, the trailing `Z`), while `oslo.log` lines are in the host's local
time without a zone. On a host whose time zone is not UTC, the same event appears with different hours in
`ovs-vswitchd.log` and in `neutron-openvswitch-agent.log`.

## Validity and limits

- Written from the 2026.1 sources of `oslo.log`, `oslo.context`, `oslo.middleware` and Kolla-Ansible, and from the
  Nova and Kolla-Ansible documentation linked above. Checked on an all-in-one Kolla-Ansible 2026.1 lab (ML2/OVS, no
  HAProxy, host in UTC) on 2026-10-27: the log directories and file names of the table (plus `nova-metadata.log`,
  `nova-novncproxy.log`, `privsep-helper.log`, `dnsmasq.log`, `mariadb-bootstrap.log`, `ansible.log`), the commands
  of *Where the logs are* (with `sudo`), debug on and off for `nova_compute` (`systemctl restart` returned after 62 s),
  and the GMR. Real lines seen there, redacted:

  ```text
  2026-10-27 13:15:17.272 7 INFO nova.compute.manager [-] [instance: <server-id>] VM Stopped (Lifecycle Event)
  [pid: 27|app: 0|req: 175/696] 192.0.2.10 () {28 vars in 305 bytes} [Tue Oct 27 13:20:12 2026] GET / => generated 377 bytes in 3 msecs (HTTP/1.1 200) 3 headers in 91 bytes (1 switches on core 0)
  2026-10-27T13:15:46.733Z|00844|connmgr|INFO|br-int<->tcp:127.0.0.1:6633: 8 flow_mods in the 4 s starting 42 s ago ...
  2026-10-27 13:19:25.681480+00:00 [info] <0.395310.0> connection 192.0.2.10:35334 -> 192.0.2.10:5672 - nova-compute ...
  2026-10-27 13:19:16.682+0000: 5159: error : virQEMUCapsCacheLookupDefault:5959 : invalid argument: ...
  2026-10-27 13:09:40 5952 [Warning] Aborted connection 5952 to db: 'heat' user: 'heat' host: ...
  ```

  (`nova-compute.log`, `nova-api-uwsgi.log`, `ovs-vswitchd.log`, `rabbit@<host>.log`, `libvirtd.log`,
  `mariadb.log`.) The RabbitMQ and libvirtd lines carry their offset (`+00:00`, `+0000`), the MariaDB line does not.
- File names are those of Kolla-Ansible 2026.1 defaults. Earlier releases ran some APIs differently (no `-uwsgi.log`
  file), and a deployment can override `log_file`, the format strings, or the container names.
- For package installs, only the Nova file names and unit names come from upstream documentation; the other
  projects' paths depend on the distribution's packaging.
- The global request id is set only on the calls where the calling code passes its context; this page does not
  list which inter-service calls do.
- The HAProxy file name (`haproxy_latest`) comes from the `fluentd` role defaults; the exact file name on disk
  (with fluentd's file output suffix) was not checked.
- The GMR was checked for `nova-compute` only, not for a service under uWSGI.
- Each instance's QEMU log is inside the `nova_libvirt` container: `sudo docker exec nova_libvirt tail
  /var/log/libvirt/qemu/<instance-name>.log` worked on the lab.

## Related

- [What a VM creation traverses, and how to trace one failed boot by request id](../diagnostics/what-a-vm-creation-traverses-and-tracing-by-request-id.md)
- [The logs of one compute do not line up with the controllers: clock skew](../diagnostics/compute-clock-skew.md)
- [Logs index](README.md)
- oslo.log configuration options: https://docs.openstack.org/oslo.log/latest/configuration/index.html
- Kolla-Ansible central logging guide: https://docs.openstack.org/kolla-ansible/latest/reference/logging-and-monitoring/central-logging-guide.html

---

*Provenance: written by agent:ozarland-author, from upstream source code and documentation (2026.1).
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Fwhere-openstack-logs-are-and-how-to-read-a-line.md&title=Feedback%3A+logs%2Fwhere-openstack-logs-are-and-how-to-read-a-line.md):
your release, your deployment method, and what happened.
