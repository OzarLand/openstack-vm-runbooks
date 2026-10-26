---
title: The API returns 503, an HAProxy backend is DOWN, or the VIP flaps or sits on two nodes
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [haproxy, keepalived, keystone, nova, neutron]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers in the loadbalancer group
  network: any
  storage: any
  os: any
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/high-availability/haproxy-guide.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/multinode.html
  - https://docs.haproxy.org/2.8/configuration.html
  - https://docs.haproxy.org/2.8/management.html
  - https://keepalived.readthedocs.io/en/latest/configuration_synopsis.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# The API returns 503, an HAProxy backend is DOWN, or the VIP flaps or sits on two nodes

On Kolla-Ansible, every API call goes to a virtual IP (VIP) that keepalived holds on one node of the
`loadbalancer` group. HAProxy on that node sends the call to the service containers on the controllers, and only to
those whose health check passes. This page covers the three ways this front door fails: HAProxy has no healthy
backend left (503), one backend is DOWN (no error, less capacity), and keepalived moves the VIP back and forth or
puts it on two nodes at once.

## Symptom

### 503 from one API, or from all of them

The client gets a 503 from HAProxy itself, not from the service:

```text
$ openstack server list
HttpException: 503: Server Error for url: https://<internal-vip-fqdn>:8774/v2.1/servers/detail, 503 Service Unavailable: No server is available to handle this request.
```

This is how openstacksdk words it (`openstack/exceptions.py`: the HTML body is stripped of its tags and joined
with colons). Commands that go through keystoneauth only print `Service Unavailable (HTTP 503)`. The body comes from
HAProxy (`src/http.c`), and `curl` shows it directly:

```text
$ curl -si https://<internal-vip-fqdn>:8774/
HTTP/1.1 503 Service Unavailable
...
<html><body><h1>503 Service Unavailable</h1>
No server is available to handle this request.
</body></html>
```

If the 503 comes from Keystone (port 5000), every command fails, because none can get a token.

On the node that holds the VIP, the HAProxy log (**Kolla-Ansible:** `/var/log/kolla/haproxy/`, the `haproxy_latest`
file that fluentd writes from HAProxy's syslog) shows each backend server going DOWN, then the backend left empty.
The format is from HAProxy's `src/server.c`, `src/check.c` and `src/backend.c`:

```text
Server nova_api_back/<controller-1> is DOWN, reason: Layer7 timeout, check duration: <n>ms. 2 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Server nova_api_back/<controller-2> is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: <n>ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Server nova_api_back/<controller-3> is DOWN, reason: Layer7 wrong status, ... 0 active and 0 backup servers left. ...
backend nova_api_back has no server available!
```

Only the message is shown: fluentd stores each syslog line as a record, with the host and the `haproxy` process
name around it. The backend is `<service>_back`, named after the service's key in the role defaults (`nova_api`,
`keystone_internal`, `placement_api`, ...). The `info` text after `Layer4 connection problem` is the system error.

### One backend DOWN, no user-visible error

Users see nothing, or slower answers. The HAProxy log has one `Server <service>_back/<controller> is DOWN` line,
with `1 active` or `2 active ... servers left`. The stats socket shows the server `DOWN` (see *Quick checks*).

### VIP flapping, or on two nodes

API calls fail for a few seconds at a time, existing connections are reset, or some clients reach the APIs and
others do not. The keepalived log of the load balancer nodes (**Kolla-Ansible:** `sudo docker logs keepalived`; the
container runs keepalived in the foreground with logging to the console) shows repeated transitions. The messages
are from keepalived's `vrrp.c`, `vrrp_scheduler.c` and `vrrp_track.c`; the instance is named
`kolla_internal_vip_<virtual-router-id>`:

```text
(kolla_internal_vip_51) Entering MASTER STATE
(kolla_internal_vip_51) Entering BACKUP STATE
Script `check_alive` now returning 1
VRRP_Script(check_alive) failed (exited with status 1)
(kolla_internal_vip_51) Entering FAULT STATE
(kolla_internal_vip_51) Receive advertisement timeout
```

When another VRRP cluster uses the same virtual router ID on the same layer 2 segment:

```text
(kolla_internal_vip_51) received an invalid passwd from <other-node-address>!
(kolla_internal_vip_51) ip address associated with VRID 51 not present in advert from <other-node-address>: <internal-vip>
```

## Impact

- 503 on Keystone: the whole control plane is unusable for users and for services that need new tokens.
- 503 on one service: that API is down (no new instance if Nova or Neutron, no attach if Cinder).
- One backend DOWN: no error, less capacity on the remaining backends.
- VIP flapping: short outages at each move, connections through the VIP reset, including the services' own
  database connections, which go through the internal VIP to ProxySQL. RabbitMQ is not affected: services connect
  to the RabbitMQ nodes directly. Metadata requests (cloud-init at boot) go through the internal VIP too, and fail
  during the outage.
- VIP on two nodes: clients reach whichever node answered their ARP last; behaviour is erratic until one node
  drops the address.
- In every case, running instances keep running and keep their network: their data path does not go through
  HAProxy. Only instances booting at that moment can miss their metadata.

## Quick checks

Read-only. Run on each node of the `loadbalancer` group.

```bash
# 1. Which node holds the VIP? It must be exactly one.
ip -brief address show dev <api-interface> | grep -w <internal-vip>

# 2. Are the haproxy, proxysql and keepalived containers running?
sudo docker ps --filter name=haproxy --filter name=proxysql --filter name=keepalived --format '{{.Names}} {{.Status}}'

# 3. State of every backend server, from HAProxy's stats socket (shared with the keepalived container,
#    which already has socat for its check script). Columns: proxy, server, status, seconds since last change,
#    last check status, last check HTTP code, last check text.
sudo docker exec keepalived bash -c 'echo "show stat" | socat unix-connect:/var/lib/kolla/haproxy/haproxy.sock stdio' \
  | cut -d, -f1,2,18,24,37,38,57 | grep -v -e FRONTEND | sed -e 's/,,/,-,/g' -e 's/,,/,-,/g' -e 's/,$/,-/' | column -s, -t
#    expected for a healthy service: <service>_back <controller> UP ... L7OK 200
#    a DOWN server shows L4CON (connection refused), L4TOUT, L7TOUT (no answer to the HTTP check) or L7STS (wrong code)

# 4. Which API answers 503, and when: every request is logged (option httplog). A 503 that HAProxy produced
#    itself, with no server left, shows <NOSRV> as the server and termination flags starting with SC.
sudo grep -h -e "<NOSRV>" -e " 503 " /var/log/kolla/haproxy/haproxy_latest* | tail -20

# 5. The last HAProxy and keepalived events
sudo grep -h -e "is DOWN" -e "is UP" -e "no server available" /var/log/kolla/haproxy/haproxy_latest* | tail -20
sudo docker logs --since 2h keepalived 2>&1 | grep -E "Entering|FAULT|VRRP_Script|now returning|invalid passwd|not present in advert" | tail -20
```

Then, for a DOWN server, run its health check by hand from the load balancer node, against the controller's own
address and port (not the VIP):

```bash
curl -s -o /dev/null -w '%{http_code} %{time_total}\n' --max-time 10 http://<controller-api-address>:<listen-port><check-path>
# Kolla-Ansible 2026.1 checks: keystone, neutron, glance, cinder GET /healthcheck; nova, placement GET /;
# all expect status 200. Use https:// when the backend uses TLS (kolla_enable_tls_backend).
```

## Diagnosis

### How Kolla-Ansible's HAProxy decides a backend is DOWN

Each API backend server line carries `check inter 2000 rise 2 fall 5` (`haproxy_health_check` in the
`haproxy-config` role): one check every 2 s, DOWN after 5 failures in a row (about 10 s), UP again after 2 successes.
The OpenStack API backends add `option httpchk GET /healthcheck` (or `GET /` for Nova and Placement) and
`http-check expect status 200` in each role's defaults. A check times out after `timeout check 10s`
(`haproxy_check_timeout`).

So a server is DOWN when its container does not listen (`Layer4 connection problem`), accepts but does not answer the
HTTP check within 10 s (`Layer7 timeout`: all workers busy, stuck on the database), or answers something other than
200 (`Layer7 wrong status`). When every server of a backend is DOWN, HAProxy answers 503 itself.

Every frontend also gets `option httplog` (HTTP services) or `option tcplog` (TCP services) from
`haproxy_frontend_http_extra` and `haproxy_frontend_tcp_extra` in the `haproxy-config` role defaults, so every
request is logged with its frontend (`<service>_front`), backend and server, timers, status code and termination
flags. A 503 that HAProxy answered itself because no server was left has `<NOSRV>` as the server name (HAProxy
configuration manual, section 8.2.3); a 503 from the service itself names a real server.

### How keepalived moves the VIP

The `vrrp_instance` in `keepalived.conf` has every node in `state BACKUP` with `nopreempt`, priorities 1, 2, 3 by
the node's position in the `loadbalancer` group, advertisements every second (`advert_int 1`), and
`track_script check_alive`. `/check_alive.sh`, from the Kolla `keepalived` image, runs every script in `/checks` every 2 s
and fails if any of them fails. Kolla-Ansible puts two there by default:

- `check_alive_haproxy.sh`: `echo "show info" | socat unix-connect:/var/lib/kolla/haproxy/haproxy.sock stdio`;
- `check_alive_proxysql.sh`: `SELECT 1` on ProxySQL's admin socket (ProxySQL is enabled by default in 2026.1,
  `enable_proxysql: "{{ enable_mariadb }}"`).

After 2 failures (`fall 2`) the instance goes to FAULT and releases the VIP, and it needs 10 successes (`rise 10`)
to leave FAULT. So a stuck ProxySQL moves the VIP as surely as a stuck HAProxy. Keepalived's log says only
`VRRP_Script(check_alive) failed`; to know which check failed, run each by hand on that node:

```bash
sudo docker exec keepalived /checks/check_alive_haproxy.sh; echo $?
sudo docker exec keepalived /checks/check_alive_proxysql.sh; echo $?
# 0 is healthy, 1 is the failing daemon. /check_alive.sh itself prints "Keepalived check script <path> failed." for each.
```

A backup takes the VIP when it stops receiving the master's advertisements (`Receive
advertisement timeout`). With `nopreempt`, the VIP does not move back when the old master recovers.

Advertisements go by multicast by default (`keepalived_traffic_mode: "multicast"`), IP protocol 112, or by unicast
to the other nodes when `keepalived_traffic_mode: "unicast"`.

## Possible causes

| What you see | Cause |
|---|---|
| 503, every server of one backend DOWN with `L4CON` | Cause 1: the service is down on every controller |
| 503, servers DOWN with `L7TOUT` or `L7STS` | Cause 2: the service runs but cannot answer its health check |
| One server DOWN, the others UP | Cause 3: one controller's container or network |
| No 503 but connection refused or timeout on the VIP; VIP on no node | Cause 4: HAProxy down on every load balancer node, every keepalived in FAULT |
| VIP moves every few minutes, `VRRP_Script(check_alive) failed` | Cause 5: HAProxy or ProxySQL failing or restarting on the master |
| VIP moves, `Receive advertisement timeout` without script failures | Cause 6: advertisements lost between the nodes |
| VIP on two nodes at once | Cause 6; more rarely cause 7, another VRRP cluster with the same virtual router ID and the same password |

### Cause 1: the service is down on every controller

`docker ps` shows the service's container stopped or restarting on every controller. Its own log
(`/var/log/kolla/<service>/`) says why it exited: configuration error after a change, database unreachable at
start-up.

### Cause 2: the service runs but its health check fails

The container is up and listens, but the check path does not answer 200 within 10 s. Usual reasons: workers stuck on
a dependency (MariaDB, RabbitMQ, memcached for Keystone), all workers busy under load, or a TLS mismatch between
HAProxy and the backend after a certificate change (the `check-ssl` check fails before the HTTP request, with a `Layer6` or socket
reason). The manual `curl` of *Quick checks* reproduces it.

### Cause 3: one controller

One server DOWN: that controller's container is stopped, or the load balancer node cannot reach that controller's
API address. Nothing breaks for users while the other servers are UP.

### Cause 4: HAProxy down on every load balancer node

The `haproxy` container is stopped or restarting everywhere (a configuration error after `kolla-ansible
reconfigure`, a certificate file missing). Each keepalived's `check_alive` fails, every instance goes to FAULT, and
no node holds the VIP. Clients get connection refused or a timeout, not a 503.

### Cause 5: HAProxy or ProxySQL unstable on the master

`check_alive` fails on the master because one of its checks fails: the HAProxy socket does not answer (the `haproxy`
container restarts in a loop, or is overloaded), or ProxySQL's admin socket does not answer (the `proxysql`
container is stuck or restarting). Keepalived goes to FAULT, the VIP moves, and may come back later. Run the two
checks by hand (*Diagnosis*) before restarting anything: restarting a healthy HAProxy on the master cuts every API
connection and moves the VIP again.

### Cause 6: VRRP advertisements lost

The backups stop hearing the master: a firewall dropping IP protocol 112, a switch filtering multicast, or the API
network itself flapping. Each backup that times out takes the VIP: it flaps, or two nodes hold it. On each node,
`sudo tcpdump -ni <api-interface> ip proto 112` shows which advertisements arrive (one per second from the master).

### Cause 7: virtual router ID collision

Another keepalived cluster (another OpenStack, a database cluster) on the same layer 2 segment uses the same
`virtual_router_id` (Kolla-Ansible default `keepalived_virtual_router_id: "51"`). If its password differs, keepalived
drops its advertisements and logs `received an invalid passwd from <other-node-address>!`; the VIP is not split, but
the log fills and a change on either side can make it worse. If the passwords match, keepalived accepts the other
cluster's advertisements, logs `ip address associated with VRID 51 not present in advert`, and elections between
the two clusters can leave the VIP on two nodes or on none. The Kolla-Ansible multinode guide says this ID must be
unique on the layer 2 network.

## Resolution

### For causes 1 and 3: start the service again, after reading why it stopped

Read the service's log on that controller first: a container that exits at start-up will exit again.

> ⚠️ **State-changing.** Restarts the service's container on one controller. Requests in flight on that controller
> fail; the other backends keep serving. Precondition: the cause of the exit is understood and fixed. Undo: none
> needed.

```bash
sudo docker restart <container>          # e.g. nova_api, keystone, neutron_server
```

HAProxy marks the server UP again after 2 successful checks (about 4 s). If a configuration change broke the
service, fix it in `/etc/kolla/` on the deploy host and redeploy that service with `kolla-ansible reconfigure -t
<service>`.

### For cause 2: fix the dependency, not HAProxy

The backend is DOWN because the service cannot answer. Fix what it waits on: see
[VM creation fails platform-wide: check the shared dependencies first](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md)
and, for Keystone, [expired certificates or Fernet keys](../diagnostics/expired-certificates-and-fernet-keys-break-every-api-call.md).
Restarting the service while its database is unreachable only loses the requests in flight.

### For causes 4 and 5: bring HAProxy or ProxySQL back on the load balancer nodes

Restart only the daemon whose check fails (*Diagnosis*). Read why it stops first: `sudo docker logs --tail 50 haproxy`
shows configuration errors (`[ALERT]` lines with the file and line) and missing certificate files;
`sudo docker logs --tail 50 proxysql` and `/var/log/kolla/proxysql/` show ProxySQL's.

If the rendered configuration is wrong (a change in `/etc/kolla/`, a missing certificate), fix it on the deploy host
and run `kolla-ansible reconfigure -i <inventory> -t loadbalancer`, which renders the files and restarts the
containers itself. A plain restart is for a transient failure, when the configuration on the node is already right:

> ⚠️ **State-changing.** Restarts HAProxy (or ProxySQL) on one node. If this node holds the VIP, every API connection
> (or every database connection) through it is cut, and keepalived may move the VIP while the daemon is down.
> Precondition: its check fails on this node and the configuration on the node is correct. Do one node at a time.
> Undo: none needed.

```bash
sudo docker restart haproxy      # or: sudo docker restart proxysql
```

On a node in FAULT, keepalived leaves FAULT after 10 successful checks (`rise 10`, about 20 s): `VRRP_Script(check_alive)
succeeded`, then `Entering BACKUP STATE`, or `MASTER` if no other node holds the VIP.

### For causes 6 and 7: one VIP holder, then a working VRRP

If two nodes hold the VIP, stop keepalived on the one that should not have it. Keepalived removes its VIPs when
it stops.

> ⚠️ **State-changing.** Stops keepalived on one node: it releases the VIP and stops taking part in failover.
> Connections that reached this node are reset. Precondition: another node holds the VIP and its HAProxy has its
> backends UP (*Quick checks* 1 and 3 there). Undo: `sudo docker start keepalived`.

```bash
sudo docker stop keepalived
```

Then fix the transport of the advertisements, and only then start it again:

- cause 6: allow IP protocol 112 between the load balancer nodes on the API interface, or have the network team
  allow the VRRP multicast group on that segment; or switch to unicast, which needs no multicast;
- cause 7: choose a virtual router ID not used on that segment.

Both are deployment settings in `/etc/kolla/globals.yml`:

```yaml
keepalived_traffic_mode: "unicast"        # cause 6, if multicast cannot be fixed
keepalived_virtual_router_id: "<unused-id>"   # cause 7, 0 to 255, unique on the layer 2 segment
```

> ⚠️ **State-changing.** Rewrites `keepalived.conf` and restarts keepalived on every load balancer node. With a new
> virtual router ID or traffic mode, the backups restart before the master, so for a moment nodes with the new
> settings do not hear the master with the old ones and take the VIP: it is briefly on two nodes, then settles on
> one. Plan a short API interruption. Precondition: every node has the same settings. Undo: put the old values back
> and run the same command.

```bash
kolla-ansible reconfigure -i <inventory> -t loadbalancer
```

## Verification

```bash
# Exactly one node holds the VIP (run on each loadbalancer node, expect one match in total)
ip -brief address show dev <api-interface> | grep -w <internal-vip>

# Every backend server UP
sudo docker exec keepalived bash -c 'echo "show stat" | socat unix-connect:/var/lib/kolla/haproxy/haproxy.sock stdio' \
  | cut -d, -f1,2,18 | grep -v -e FRONTEND -e BACKEND | grep -v ',UP$'
# expected: only the header line

# No transitions in the last minutes
sudo docker logs --since 10m keepalived 2>&1 | grep -E "Entering|FAULT"
# expected: nothing

# The APIs answer through the VIP
openstack token issue -f value -c expires
openstack server list --limit 1
```

## Validity and limits

- Written from Kolla-Ansible 2026.1: the `loadbalancer` role (`keepalived.conf.j2`, `check_alive_haproxy.sh.j2`,
  `check_alive_proxysql.sh.j2`, `haproxy_main.cfg.j2`, `keepalived.json.j2`, `tasks/config.yml`, defaults), the
  `haproxy-config` role (`haproxy_single_service_split.cfg.j2`, `haproxy_health_check`,
  `haproxy_frontend_http_extra`), `group_vars/all/proxysql.yml`, the Kolla 2026.1 `keepalived` image
  (`docker/keepalived/check_alive.sh`), the health checks in the `keystone`, `nova`,
  `neutron`, `glance`, `cinder` and `placement` role defaults, the `fluentd` role defaults for the HAProxy log, and
  the HAProxy and multinode guides. HAProxy messages from the HAProxy 2.8 source (`src/server.c`, `src/check.c`,
  `src/backend.c`, `src/http.c`) and its configuration and management manuals. Keepalived messages from the
  keepalived source (`keepalived/vrrp/`). Client messages from keystoneauth1 and openstacksdk. Not reproduced on a
  lab.
- The HAProxy and keepalived versions in your images may differ from the sources read; the messages quoted have been
  stable for several major versions, but check yours. The `show stat` column numbers are those of HAProxy 2.8's
  management guide.
- Kolla-Ansible only. Other deployment tools use other check paths, check intervals and VRRP settings; the
  diagnosis applies, the commands and file paths do not.
- The commands use `docker`. With `kolla_container_engine: podman`, replace `sudo docker` with `sudo podman`; the
  container names and paths are the same.
- Not covered: the external VIP on a separate interface (`track_interface`), ProxySQL's own failures beyond its
  keepalived check (its backends, its query rules), HAProxy with `haproxy_single_external_frontend`, and Let's
  Encrypt.

## Related

- [Every API call fails at once: expired certificates or Fernet keys](../diagnostics/expired-certificates-and-fernet-keys-break-every-api-call.md)
- [VM creation fails platform-wide: check the shared dependencies first](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md)
- [Control plane down: what still works for running instances](../diagnostics/control-plane-down-what-still-works-for-instances.md)
- Kolla-Ansible 2026.1: [HAProxy guide](https://docs.openstack.org/kolla-ansible/2026.1/reference/high-availability/haproxy-guide.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 source and documentation and the
HAProxy and keepalived source. Reviewed in its pull request (see the page history). Checked by a human operator:
not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=recovery/api-returns-503-haproxy-backend-down-or-vip-flapping.md&title=Feedback%3A+recovery/api-returns-503-haproxy-backend-down-or-vip-flapping.md):
your release, your deployment method, and what happened.
