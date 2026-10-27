---
title: Services fail on "Too many connections", "Max connect timeout" or "QueuePool limit". Database connections are exhausted
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [mariadb, proxysql, nova, neutron, keystone]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers, ProxySQL in front of Galera
  network: any
  storage: any
  os: any
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, 3 controllers (Galera + ProxySQL) + 2 computes, ML2/OVS", date: 2026-10-27, note: "MariaDB full (max_connections lowered on the writer: 9001 platform-wide, writer shunned) and no synced writer (one node: failover; three nodes: 9001) reproduced, KILL of one idle session run; QueuePool not reproduced (uWSGI one thread per process), ProxySQL client limit not tested, runtime max_connections raise not run (root refused on the full node; a MariaDB container restart restored it), service restart for causes 1-2 not run" }
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/databases/mariadb-guide.html
  - https://proxysql.com/documentation/global-variables/mysql-variables/
  - https://proxysql.com/documentation/stats-statistics/
  - https://docs.openstack.org/oslo.db/latest/reference/opts.html
  - https://mariadb.com/kb/en/server-system-variables/#max_connections
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Services fail on "Too many connections", "Max connect timeout" or "QueuePool limit". Database connections are exhausted

On Kolla-Ansible 2026.1, every OpenStack service reaches MariaDB through ProxySQL on the internal VIP. There are
three places where database connections can run out, each with its own message: MariaDB itself, ProxySQL (its
client side or its pool of backend connections), and the SQLAlchemy pool inside each service process. Which message
you see tells you where to look. API calls fail with HTTP 500 in all three cases (on a lab, the client got
`Bad Gateway (HTTP 502)`); running instances are not affected.

## Symptom

In the service logs (`/var/log/kolla/<service>/*.log`), one of these, usually in a traceback that ends an API
request with `500`:

```text
sqlalchemy.exc.OperationalError: (pymysql.err.OperationalError) (1040, 'Too many connections')
```

```text
sqlalchemy.exc.OperationalError: (pymysql.err.OperationalError) (9001, 'Max connect timeout reached while reaching hostgroup 0 after 10000ms')
```

```text
sqlalchemy.exc.TimeoutError: QueuePool limit of size 1 overflow 50 reached, connection timed out, timeout 30.00 (Background on this error at: https://sqlalche.me/e/20/3o7r)
```

- `1040` and `9001` are MySQL protocol errors; oslo.db does not treat them as connection errors (its connection
  filter matches 2002, 2003, 2006, 2013 and 1047), so it does not retry them. On a lab, `9001` reached the logs as
  `sqlalchemy.exc.OperationalError` (237 lines), and as `oslo_db.exception.DBConnectionError` in two nova-conductor
  service-status reports; never as `DBError`.
- Behind ProxySQL, a full MariaDB node is seen by the services as `9001`, not `1040`: the `1040` is only in
  `proxysql.log` (`Failed to mysql_real_connect() on 0:<node>:3306 , ... , 1040: Too many connections.`) and in the
  node's `mariadb.log` (`Aborted connection 0 to db: 'unconnected' user: 'unauthenticated' host: 'connecting host'
  (Too many connections)`).
- `9001` and its text come from ProxySQL (`lib/MySQL_Session.cpp`); depending on the ProxySQL version the text may
  include `for 'fast_forward' session` before `after`.
- The `QueuePool` text is SQLAlchemy's (`sqlalchemy/pool/impl.py`), raised inside one process; the numbers are that
  service's `[database] max_pool_size`, `max_overflow` and `pool_timeout`.

ProxySQL's own log (`/var/log/kolla/proxysql/proxysql.log`) has, when its client limit is reached:

```text
... [WARNING] mysql-max_connections reached. Returning 'Too many connections'
```

## Impact

API requests that need the database fail with 500, for the services concerned or all of them. Workers and agents
retry or log errors; computes may be reported `down` if the conductor cannot write their heartbeat. Running instances
keep running. Killing database sessions (below) rolls back their open transactions.

## Quick checks

Read-only. On a controller.

```bash
# 1. Which message, which services, since when (the glob needs root: /var/log/kolla is not readable by others)
sudo sh -c 'grep -l -e "Too many connections" -e "Max connect timeout" -e "QueuePool limit" /var/log/kolla/*/*.log'
sudo sh -c 'grep -h -e "Too many connections" -e "Max connect timeout" -e "QueuePool limit" /var/log/kolla/*/*.log' | tail -5

# 2. MariaDB: limit, current, peak, and who holds the connections (asks for the database root password,
#    database_password in /etc/kolla/passwords.yml on the deploy host)
sudo docker exec -it mariadb mariadb -u root -p -e "
  SHOW GLOBAL VARIABLES LIKE 'max_connections';
  SHOW GLOBAL STATUS WHERE Variable_name IN ('Threads_connected','Max_used_connections','Aborted_connects','Connection_errors_max_connections');
  SELECT user, SUBSTRING_INDEX(host, ':', 1) AS client, command, COUNT(*) AS n
    FROM information_schema.processlist GROUP BY user, client, command ORDER BY n DESC LIMIT 20;"

# 3. ProxySQL: client connections, backend pool per server, connect timeouts (asks for proxysql_admin_password;
#    the keepalived container has the client and the admin socket, as its ProxySQL check uses them). Run it on the
#    controller that holds the internal VIP: the services connect to that ProxySQL, the others show 0 connections.
sudo docker exec -it keepalived mariadb --no-defaults --protocol socket --socket=/var/lib/kolla/proxysql/admin.sock \
  --user=kolla-admin -p --skip-ssl -e "
  SELECT Variable_Name, Variable_Value FROM stats_mysql_global
   WHERE Variable_Name IN ('Client_Connections_connected','Client_Connections_aborted','Server_Connections_connected','max_connect_timeouts');
  SELECT hostgroup, srv_host, status, ConnUsed, ConnFree, ConnERR, MaxConnUsed FROM stats_mysql_connection_pool;"
```

Expected on a healthy platform: `Threads_connected` far below `max_connections` (10000 in Kolla-Ansible's
`galera.cnf.j2`), `Connection_errors_max_connections` not growing, ProxySQL backends `ONLINE` with `ConnUsed` far
below 10000, `max_connect_timeouts` not growing.

## Diagnosis

### The three limits in a default Kolla-Ansible 2026.1

| Where | Setting | Default | What you see when it is reached |
|---|---|---|---|
| MariaDB, per node | `max_connections` (`galera.cnf.j2`) | 10000 | `1040 Too many connections` from MariaDB, to ProxySQL and to anyone connecting directly |
| ProxySQL, client side | `mysql-max_connections` (`proxysql_max_connections`) | 40000 | `1040 Too many connections` from ProxySQL, and the `mysql-max_connections reached` warning |
| ProxySQL, per backend | `max_connections` of each server (`proxysql_backend_max_connections`) | 10000 | queries wait for a backend connection, then `9001 Max connect timeout ... after 10000ms` (`connect_timeout_server_max`, 10000 ms) |
| Each service process | `[database] max_pool_size` + `max_overflow`, wait `pool_timeout` | Kolla-Ansible: `max_pool_size = 1` (`database_max_pool_size`); `max_overflow = 1000` for Nova, oslo.db's 50 elsewhere; SQLAlchemy's 30 s wait | `QueuePool limit of size ... reached` in that service only |

### Telling them apart

- **`QueuePool limit` in one service**, the database itself fine (check 2 and 3 normal): the problem is inside that
  service. Its processes hold their connections: long transactions, requests stuck on something else (RabbitMQ,
  another API) while holding a session, or many concurrent requests per process. Other services are fine.
- **`9001 Max connect timeout`**: ProxySQL could not get a backend connection in 10 s. Either the backend servers
  are at their `ConnUsed` limit (check 3), or no node is `ONLINE` in the writer hostgroup: with
  `mysql_galera_hostgroups`, ProxySQL moves a Galera node that is not synced out of the writer hostgroup (to the
  offline hostgroup, `3` for the first shard), and with no writer left every query times out like this. If check 3
  shows no `ONLINE` server in hostgroup `0`, see [Galera lost quorum](galera-cluster-lost-quorum-database-unavailable.md).
- **`1040 Too many connections`**, ProxySQL's warning present: the client side of ProxySQL is full (40000). Look at
  `Client_Connections_connected` and at which services keep connections open.
- **`1040 Too many connections`**, no ProxySQL warning, `Threads_connected` near `max_connections` on a MariaDB
  node: MariaDB itself is full. ProxySQL's backend limit equals MariaDB's, so this needs other clients on top of
  ProxySQL's: direct connections, backups, monitoring, or a ProxySQL limit raised above MariaDB's.
- **MariaDB full behind ProxySQL** shows as `9001`, not `1040` (see *Symptom*). ProxySQL keeps serving from the
  backend connections it already holds; when it needs new ones, it logs `1040: Too many connections.` for them, then
  `Shunning server <node>:3306 with 5 errors/sec. Shunning for 10 seconds`, drops the node's free connections, and
  with the writer shunned every service fails on `9001`. Check 3 then shows the writer `SHUNNED`, `ConnFree` 0 and
  `ConnERR` growing.

### Who holds the connections

The processlist query of check 2 groups by user and client address: one OpenStack service user (`nova`, `neutron`,
...) with a count far above the others, or many `Sleep` connections from one address, points to the service and the
host. Behind ProxySQL, the client address MariaDB sees is the ProxySQL node; use ProxySQL's
`stats_mysql_processlist` table to see the real clients.

## Possible causes

| What you see | Cause |
|---|---|
| `QueuePool limit` in one service only | Cause 1: the service's pool is too small for its load, or its sessions are held too long |
| `9001`, backends `ONLINE` with `ConnUsed` at the limit | Cause 2: one service, or a few, open far more connections than usual (a burst, a leak, a restart storm) |
| `9001`, no server `ONLINE` in the writer hostgroup | Cause 3: Galera is not synced: not a connection problem |
| `1040` from ProxySQL or from MariaDB | Cause 2, at the client side; or limits changed so that ProxySQL allows more than MariaDB |

## Resolution

### For cause 1: the pool of one service

Find why the service holds its connections (its own log, the other errors around the `QueuePool` lines). If it
waits on another dependency, fix that dependency. A restart releases the pool at once:

> ⚠️ **State-changing.** Restarts the service's container on one controller. Its requests in flight fail; the other
> controllers keep serving through HAProxy. Precondition: the reason it held its connections is understood, or
> this is a stop-gap while you look. Undo: none needed.

```bash
sudo systemctl restart kolla-<container>-container.service   # e.g. kolla-nova_api-container.service
```

Raising `max_pool_size` or `max_overflow` is a configuration change for that service in `/etc/kolla/config/`, rolled
out with `kolla-ansible reconfigure -t <service>`. Do it only once the load is understood: a larger pool per process,
times the number of processes and controllers, can push ProxySQL or MariaDB to their limits instead.

### For cause 2: the service that opened too many connections

Identify it (check 2, or `stats_mysql_processlist` in ProxySQL), then restart it as above, one controller at a time.
That closes its connections.

If MariaDB is the full one and nothing else can be done quickly, its limit can be raised at runtime:

> ⚠️ **State-changing.** Raises `max_connections` on one MariaDB node until its next restart. Each connection costs
> memory on that node. Precondition: the node has the memory for it, and you can still connect to it: on a lab, a
> node at its limit refused root too (`ERROR 2002 (HY000): Received error packet before completion of TLS handshake.
> ... 1040 - Too many connections`), and the only way back seen was
> `sudo systemctl restart kolla-mariadb-container.service` on that node (it rejoined by IST, `Synced` 33 s after
> the restart, the other two nodes kept serving). Undo: `SET GLOBAL max_connections = 10000;`.
> To keep it, set it in `/etc/kolla/config/galera.cnf` and run `kolla-ansible reconfigure -t mariadb`, which restarts
> the MariaDB nodes one by one.

```bash
sudo docker exec -it mariadb mariadb -u root -p -e "SET GLOBAL max_connections = <new-limit>;"
```

As a last resort, sessions can be killed:

> 🛑 **Destructive.** Kills database sessions. What can be lost: the open transaction of each killed session is
> rolled back, and the request that owned it fails; if the service does not handle that well, an instance, a port or
> a volume can be left half-updated. Who is affected: the users of the requests in flight on those sessions.
> Precondition: the sessions are idle (`Sleep`) for long (`time` column of `information_schema.processlist`) and
> belong to the service you identified; restarting that service, which closes them cleanly, was not possible. No
> undo.

```bash
sudo docker exec -it mariadb mariadb -u root -p -e "
  SELECT id, user, host, command, time FROM information_schema.processlist
   WHERE user = '<service-user>' AND command = 'Sleep' AND time > <seconds> ORDER BY time DESC;"
sudo docker exec -it mariadb mariadb -u root -p -e "KILL <id>;"
```

### For cause 3: Galera, not connections

Do not raise limits: follow [Galera lost quorum](galera-cluster-lost-quorum-database-unavailable.md). When a node is
synced again, ProxySQL puts it back `ONLINE` by itself.

## Verification

```bash
# No new errors in the last minutes (the lines of all files are not in time order: check their timestamps)
sudo sh -c 'grep -h -e "Too many connections" -e "Max connect timeout" -e "QueuePool limit" /var/log/kolla/*/*.log' | tail -3
# MariaDB well below its limit, ProxySQL backends ONLINE with spare connections (checks 2 and 3 again)
# The APIs answer
openstack server list --limit 1
openstack network list --limit 1
```

## Validity and limits

- Written from Kolla-Ansible 2026.1 (`mariadb` role `galera.cnf.j2`; `loadbalancer` role `proxysql.yaml.j2`, its
  defaults and `check_alive_proxysql.sh.j2`; `group_vars/all/database.yml`; the `[database]` sections of
  `nova.conf.j2`, `neutron.conf.j2` and `keystone.conf.j2`), ProxySQL's source (`lib/MySQL_Session.cpp` for the
  `9001` and `Too many connections` errors, `lib/MySQL_Thread.cpp` for `connect_timeout_server_max`, the admin table
  definitions for `stats_mysql_connection_pool` and `stats_mysql_global`), oslo.db 18.0.0 (`options.py` defaults,
  `sqlalchemy/exc_filters.py`), SQLAlchemy 2.0.46 (`pool/impl.py`) and PyMySQL 1.1.2, the 2026.1 upper constraints.
  Partly reproduced on a lab (see *Lab observations*).
- The ProxySQL version in your image may differ from the source read; the error texts above have been stable, the
  exact wording of `9001` varies slightly.
- The ProxySQL writer hostgroup is `0` for the first shard (`writer_hostgroup = shard_id * 10`; backup writer 1,
  reader 2, offline 3, as `runtime_mysql_galera_hostgroups` showed on a lab); other shards use 10, 20...
- Kolla-Ansible with ProxySQL (the 2026.1 default when MariaDB is enabled). Without ProxySQL, HAProxy forwards to one
  Galera node and only the MariaDB and service-pool limits apply.
- Not covered: slow queries and lock waits (a different symptom: requests slow, not refused), and the `max_connections`
  of RabbitMQ.

## Lab observations

Kolla-Ansible 22.2.0 (2026.1), three controllers with Galera behind ProxySQL 3.0.11 and MariaDB 11.4, two computes,
ML2/OVS, 2026-10-27.

- **MariaDB full:** `SET GLOBAL max_connections = 20;` on the writer, which had 167 connections. API calls kept
  working for about 2.5 minutes (ProxySQL reused its 162 open backend connections); then ProxySQL logged `1040` for
  new backend connections, shunned the writer and every service failed on `9001` (nova-conductor, nova-scheduler,
  neutron, keystone, heat); the CLI got `Bad Gateway (HTTP 502)`. Root could not connect to the node to undo it;
  restarting the MariaDB container on that node restored the limit and the APIs answered 54 s later.
- **No synced writer:** `wsrep_desync = ON` and `wsrep_reject_queries = ALL` on the writer only: ProxySQL moved it to
  hostgroup 3 and made another node writer within 3 s; no service error. On all three nodes: hostgroup 0 empty, every
  service on `9001`; after the undo ProxySQL put the nodes back by itself.
- **Kill:** killing one idle `neutron` session closed the matching ProxySQL client connection (`Closing unhealthy
  client connection ... user 'neutron'`); the next API calls succeeded.
- **Not reproduced:** the `QueuePool limit` message. With `max_overflow = 0` and `pool_timeout = 5` on one
  neutron-server, 2000 concurrent API requests gave none: Kolla's uWSGI runs 4 processes with one thread each, so an
  API process never shares its pool. ProxySQL's client limit (40000) and its `mysql-max_connections reached` warning
  not tested; raising `max_connections` at runtime not run on a full node (the connection was refused), only its undo
  value set after the restart.

## Related

- [Every API fails on database errors. The Galera cluster lost quorum](galera-cluster-lost-quorum-database-unavailable.md)
- [VM creation fails platform-wide: check the shared dependencies first](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md)
- [The API returns 503, an HAProxy backend is DOWN](api-returns-503-haproxy-backend-down-or-vip-flapping.md): a
  ProxySQL that stops answering also moves the VIP.

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 source, the ProxySQL source, and the
oslo.db and SQLAlchemy versions of the 2026.1 upper constraints. Reviewed in its pull request (see the page
history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=recovery/database-too-many-connections-or-pool-exhausted.md&title=Feedback%3A+recovery/database-too-many-connections-or-pool-exhausted.md):
your release, your deployment method, and what happened.
