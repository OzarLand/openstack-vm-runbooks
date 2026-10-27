---
title: Reading the HAProxy, keepalived and memcached logs in front of the OpenStack APIs
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [haproxy, keepalived, proxysql, memcached, keystone]
deployment_methods: [kolla-ansible, any]
environment:
  topology: multinode, 3 controllers behind a virtual IP
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by: []
upstream_refs:
  - https://docs.haproxy.org/3.0/configuration.html
  - https://github.com/haproxy/haproxy/tree/v3.0.0/src
  - https://github.com/acassen/keepalived/tree/master/keepalived/vrrp
  - https://github.com/memcached/memcached/blob/master/doc/protocol.txt
  - https://opendev.org/openstack/kolla-ansible/src/branch/stable/2026.1/ansible/roles/loadbalancer
  - https://opendev.org/openstack/kolla-ansible/src/branch/stable/2026.1/ansible/roles/haproxy-config/defaults/main.yml
  - https://opendev.org/openstack/kolla-ansible/src/branch/stable/2026.1/ansible/roles/memcached/templates/memcached.json.j2
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Reading the HAProxy, keepalived and memcached logs in front of the OpenStack APIs

Every API call goes through a virtual IP held by keepalived, to HAProxy, to one API backend; Keystone and the
`keystonemiddleware` of every service use memcached. When the client gets a 503, a timeout or a refused connection,
these three logs say whether the request reached a backend at all. Messages are quoted from the HAProxy 3.0 source
and documentation, the keepalived source (`keepalived/vrrp/`), and memcached's protocol documentation; how
Kolla-Ansible 2026.1 configures them is read from its `loadbalancer`, `haproxy-config` and `memcached` roles.

## HAProxy

### Where the log is

HAProxy sends its log through syslog. **Kolla-Ansible** points it at fluentd (`log <syslog_server>:<port> local1` in
`haproxy_main.cfg.j2`), which writes it to a file under `/var/log/kolla/haproxy/` (the `haproxy_latest` output of
the `fluentd` role, see [where the logs are](where-openstack-logs-are-and-how-to-read-a-line.md)), on each load
balancer node. **Any:** wherever your syslog daemon writes the facility HAProxy logs to.

The state now is in the stats socket (Kolla-Ansible: `/var/lib/kolla/haproxy/haproxy.sock`, read-only command):

```bash
# the keepalived container mounts the same socket and has socat (its check_alive script uses it)
echo "show stat" | sudo docker exec -i keepalived socat stdio unix-connect:/var/lib/kolla/haproxy/haproxy.sock \
  | cut -d, -f1,2,18 | column -s, -t          # proxy, server, status (UP, DOWN, MAINT)
```

### Backend state changes

When a health check changes a server's state, HAProxy logs (`src/server.c`, `src/backend.c`):

| Line (message text) | Level | What it means |
|---|---|---|
| `Server <backend>/<server> is DOWN, reason: <description>, code: <n>, info: "<text>", check duration: <n>ms. <n> active and <n> backup servers left. <n> sessions active, <n> requeued, <n> remaining in queue.` | alert | The health check failed `fall` times in a row. Kolla-Ansible's check is `check inter 2000 rise 2 fall 5`: every 2 s, down after 5 failures (about 10 s), up after 2 successes. `Backup Server …` for a backup member. |
| `Server <backend>/<server> is UP, reason: <description>, check duration: <n>ms. <n> active and <n> backup servers online. <n> sessions requeued, <n> total in queue.` | notice | The server passed `rise` checks and is back. |
| `<proxy type> <backend> has no server available!` | emergency | The last server of that backend went down. Every request to this API now gets a 503 from HAProxy itself. |

The `code` and `info` parts appear only when the check has them (`check_append_info` in `src/check.c`). The `reason` is the health check result, with HAProxy's own descriptions, for example
`Layer4 connection problem` (connection refused or reset: the service is not listening), `Layer4 timeout` (the TCP
connection did not complete in time, bounded by `timeout connect` and the check interval, so about 2 s on
Kolla-Ansible), `Layer7 timeout` (connected, but no HTTP answer within `timeout check`, 10 s on Kolla-Ansible: the
typical busy-workers case), or `Layer7 wrong status` (the HTTP check answered with an unexpected status; Kolla-Ansible
checks some APIs with `option httpchk GET /healthcheck` and expects 200).

```bash
sudo grep -E "is DOWN|is UP|has no server available" /var/log/kolla/haproxy/haproxy_latest* | tail -30
```

### Request lines

Kolla-Ansible adds `option httplog` to its HTTP frontends and `option tcplog` to its TCP frontends
(`haproxy_frontend_http_extra`, `haproxy_frontend_tcp_extra` in the `haproxy-config` role). With `option httplog`,
each request is logged at its end in HAProxy's HTTP format, documented in section 8.2.3 of the
[configuration manual](https://docs.haproxy.org/3.0/configuration.html):

```text
%ci:%cp [%tr] %ft %b/%s %TR/%Tw/%Tc/%Tr/%Ta %ST %B %CC %CS %tsc %ac/%fc/%bc/%sc/%rc %sq/%bq %hr %hs %{+Q}r
```

The fields that matter most for an API failure:

| Field | Meaning |
|---|---|
| `%b/%s` | Backend and **server** that handled the request. `<NOSRV>` means no server was chosen. |
| `%TR/%Tw/%Tc/%Tr/%Ta` | Times in ms: request received, waiting in queue, TCP connect to the server, server response time, total. `-1` means that step never completed. |
| `%ST` | HTTP status returned to the client. |
| `%tsc` | Termination state, four characters; the first two say who ended the stream and in which phase (manual section 8.5). |

Termination states worth knowing (manual section 8.5):

| State | Meaning | Typical status |
|---|---|---|
| `----` | Normal completion. | any |
| `SC--` | The server, or something between HAProxy and it, refused the connection (RST, ICMP, no route). With no server available at all, HAProxy also logs `<NOSRV>`. | 502 or 503 |
| `sC--` | `timeout connect` struck before the server accepted (10 s on Kolla-Ansible). | 503 or 504 |
| `sH--` | `timeout server` struck before the server sent its response headers: the API took longer than 1 minute (Kolla-Ansible's `haproxy_server_timeout`; some services have their own). The manual calls it "the most common anomaly". | 504 |
| `SH--` | The server closed the connection before sending its response headers: the API worker crashed or was restarted. | 502 |
| `sQ--` | The request waited too long in the queue (`timeout queue`). | 503 |
| `cD--` / `CD--` | The client went silent or aborted during the transfer. | any |

```bash
# requests that ended in 502, 503 or 504: the status follows the five timers
sudo grep -E ' -?[0-9]+/-?[0-9]+/-?[0-9]+/-?[0-9]+/-?[0-9]+ 50[234] ' /var/log/kolla/haproxy/haproxy_latest* | tail -20
```

Next step for each: `SC`/`<NOSRV>` → the backend's service and its health; `sH` → the API's own log for the same
time (the request reached it and was slow); `SH` → the API's log and its container's restarts.

## keepalived

### Where the log is

**Kolla-Ansible** runs `keepalived -nld` (`keepalived.json.j2`): in the foreground, logging to the console, so its log
is the container's output:

```bash
sudo docker logs --since 1h keepalived
```

**Any:** the journal of the `keepalived` unit, or syslog.

Kolla-Ansible's VRRP instance is named `kolla_internal_vip_<virtual_router_id>`, starts every node as `BACKUP` with
`nopreempt`, and tracks the script `check_alive` every 2 s (`interval 2`, `fall 2`, `rise 10` in
`keepalived.conf.j2`). `check_alive` is `/check_alive.sh` from the Kolla image, which **runs every script in
`/checks`** and returns non-zero if any of them fails (`docker/keepalived/check_alive.sh`). On a default 2026.1
deployment `/checks` holds:

- `check_alive_haproxy.sh`: asks HAProxy's stats socket for `show info`;
- `check_alive_proxysql.sh`: runs `SELECT 1` on ProxySQL's admin socket, because ProxySQL is enabled by default
  (`enable_proxysql: "{{ enable_mariadb }}"`).

So a failing `check_alive` means **HAProxy or ProxySQL** does not answer. Run each check by hand to know which:

```bash
sudo docker exec keepalived /checks/check_alive_haproxy.sh; echo $?
sudo docker exec keepalived /checks/check_alive_proxysql.sh; echo $?
```

### Lines

| Line (message text) | What it means |
|---|---|
| `(<instance>) Entering MASTER STATE` | This node now holds the virtual IP(s). Exactly one node should log this per failover. |
| `(<instance>) Entering BACKUP STATE` (`(init)` at start) | This node does not hold the VIP. |
| `(<instance>) Entering FAULT STATE` (`(init)` at start) | A tracked script or interface failed: this node cannot take the VIP. With Kolla-Ansible, usually `check_alive` failing because HAProxy or ProxySQL does not answer (run the checks above). |
| `Script \`check_alive\` now returning <n>` | The track script's result changed (0 = every check in `/checks` passed, non-zero = at least one failed). Read the next state change. |
| `VRRP_Script(check_alive) failed (exited with status <n>)` / `VRRP_Script(check_alive) succeeded` | Logged when the result crosses `fall` (2 failures) or `rise` (10 successes): this is when keepalived acts on it. |
| `(<instance>) Receive advertisement timeout` | A backup node stopped hearing the master's adverts and will take over: the master died, or VRRP traffic between them is lost. |
| `(<instance>) Received advert from <address> with lower priority <n>, ours <n>` | Another node advertises for the same VRID with a lower priority. |
| `(<instance>) ip address associated with VRID <id> not present in advert from <address>: <address>` | Two VRRP instances with the same `virtual_router_id` but different VIPs on the same network: usually another cluster (or another deployment) reusing the VRID. |

Sources: `keepalived/vrrp/vrrp.c`, `vrrp_scheduler.c`.

Two nodes logging `Entering MASTER STATE` without one of them logging `Entering BACKUP STATE` in between means both
hold the VIP: VRRP adverts are not getting through between them (network, firewall, multicast or unicast peers).

```bash
sudo docker logs keepalived 2>&1 | grep -E "Entering (MASTER|BACKUP|FAULT) STATE|now returning|VRRP_Script|advertisement timeout" | tail -20
```

## memcached

Kolla-Ansible runs `memcached -v … -c <memcached_connection_limit> -m <memcached_max_memory>` (`memcached.json.j2`;
defaults 5000 connections and 256 MB). With `-v` it prints errors and warnings to standard error, which on
Kolla-Ansible is `sudo docker logs memcached`. memcached says little in its log; its counters say more. They are
defined in memcached's [protocol documentation](https://github.com/memcached/memcached/blob/master/doc/protocol.txt):

| Counter | Meaning |
|---|---|
| `curr_connections` / `max_connections` | Open connections and the limit. |
| `rejected_connections` | Connections refused because the limit was reached, in `maxconns_fast` mode (memcached's default): non-zero and growing means clients are being refused. |
| `listen_disabled_num` | Times the server stopped accepting new connections at the limit; with `maxconns_fast` it can stay at 0 while connections are refused, so read `rejected_connections` first. |
| `evictions` | Valid items removed to make room. Growing fast with a small `-m`: cached tokens and data do not stay cached. |

```bash
# On a memcached node (Kolla-Ansible binds memcached to the API interface address)
# Debian/Ubuntu netcat; on RHEL-family hosts use: printf 'stats\r\nquit\r\n' | nc <api-interface-address> 11211
echo stats | nc -q1 <api-interface-address> 11211 | grep -E 'curr_connections|max_connections|listen_disabled_num|rejected_connections|evictions'
```

Keystone and the services use memcached through `oslo.cache` (`backend = oslo_cache.memcache_pool` in Kolla-Ansible's
`keystone.conf.j2` and `nova.conf.j2`) and `keystonemiddleware` (`memcached_servers`). A memcached node that stops
answering shows up in those services as slow token validation rather than as a clear error line; the Keystone page
(to come) covers what to look for there.

## Reading an API outage in order

1. **Which node holds the VIP?** keepalived's last `Entering MASTER STATE`, and `ip address show` on that node.
2. **Are HAProxy and ProxySQL up there?** Their containers, `check_alive` results in keepalived's log, and the two
   checks run by hand.
3. **Which backend lost its servers?** `is DOWN` and `has no server available!` lines, then `show stat`.
4. **What did the client get?** Request lines with the status and termination state.
5. **Then the API itself:** its own log and its `-uwsgi.log` for the same minute, on the server named in `%b/%s`.

## Validity and limits

- Messages are quoted from HAProxy 3.0 and the keepalived and memcached sources and documentation named above. The
  HAProxy and keepalived versions in a Kolla 2026.1 image come from its base distribution and may differ; these
  messages have been stable across recent versions, but the exact text was not checked against the image.
  **No line was captured on a lab.** A lab request asks for real samples.
- The HAProxy log file name under `/var/log/kolla/haproxy/` was not checked (see the first page of this folder).
- The `grep` patterns on HAProxy request lines assume the default `option httplog` format; a custom `log-format`
  changes the fields.
- Client-side symptoms of a memcached failure in Keystone and `keystonemiddleware` are not detailed here.

## Related

- [API returns 503, HAProxy backend DOWN, VIP flapping or on two nodes](../recovery/api-returns-503-haproxy-backend-down-or-vip-flapping.md) (the fix)
- [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md)
- [Every API call fails at once: certificates or Fernet keys](../diagnostics/expired-certificates-and-fernet-keys-break-every-api-call.md)
- [VM creation fails platform-wide: check the shared dependencies first](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md)
- HAProxy configuration manual, log formats (8.2) and stream state at disconnection (8.5): https://docs.haproxy.org/3.0/configuration.html
- [Logs index](README.md)

---

*Provenance: written by agent:ozarland-author, from HAProxy, keepalived, memcached and Kolla-Ansible 2026.1 source
code and documentation. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Freading-haproxy-keepalived-memcached-logs.md&title=Feedback%3A+logs%2Freading-haproxy-keepalived-memcached-logs.md):
your release, your deployment method, and what happened.
