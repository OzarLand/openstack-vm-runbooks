---
title: "Error messages: messaging and database. MessagingTimeout, AMQP connection and heartbeat lines, DBConnectionError"
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [rabbitmq, mariadb, proxysql, haproxy, nova, neutron, cinder]
deployment_methods: [any]
environment:
  topology: any
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by: []
upstream_refs:
  - https://github.com/openstack/oslo.messaging/blob/stable/2026.1/oslo_messaging/_drivers/amqpdriver.py
  - https://github.com/openstack/oslo.messaging/blob/stable/2026.1/oslo_messaging/_drivers/impl_rabbit.py
  - https://github.com/openstack/oslo.messaging/blob/stable/2026.1/oslo_messaging/rpc/client.py
  - https://github.com/openstack/oslo.db/blob/stable/2026.1/oslo_db/sqlalchemy/engines.py
  - https://github.com/openstack/oslo.db/blob/stable/2026.1/oslo_db/sqlalchemy/exc_filters.py
  - https://github.com/openstack/oslo.db/blob/stable/2026.1/oslo_db/options.py
  - https://github.com/openstack/oslo.db/blob/stable/2026.1/oslo_db/exception.py
  - https://github.com/openstack/oslo.db/blob/stable/2026.1/oslo_db/api.py
  - https://github.com/PyMySQL/PyMySQL/blob/main/pymysql/constants/CR.py
  - https://github.com/PyMySQL/PyMySQL/blob/main/pymysql/constants/ER.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/conductor/api.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/servicegroup/drivers/db.py
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Error messages: messaging and database. MessagingTimeout, AMQP connection and heartbeat lines, DBConnectionError

This page belongs to the dictionary of frequent OpenStack error messages. For each message: where it appears, what it
means, the usual causes, the next log to read, and the runbook that fixes it. The messages are the format strings of
the upstream source at `stable/2026.1`; `<…>` marks a value. Where a section shows a *Lab sample*, the lines were
captured on a Kolla-Ansible 2026.1 lab with three controllers (redacted).

How to read the RabbitMQ broker log itself is on
[reading the RabbitMQ logs](reading-rabbitmq-logs.md).

## Messaging (oslo.messaging with RabbitMQ)

### `MessagingTimeout: Timed out waiting for a reply to message ID <msg-id>`

- **Where:** in a traceback, in the log of the service that made an RPC call: `nova-api` and `nova-compute` calling
  `nova-conductor`, `nova-conductor` calling `nova-compute`, Neutron agents calling `neutron-server`, and so on. A
  second form, `Timed out waiting for a reply <reply queue> to message ID <msg-id>.`, comes from the same file
  ([`amqpdriver.py`](https://github.com/openstack/oslo.messaging/blob/stable/2026.1/oslo_messaging/_drivers/amqpdriver.py)).
- **Means:** an RPC call got no reply within `rpc_response_timeout` (60 s by default,
  [`rpc/client.py`](https://github.com/openstack/oslo.messaging/blob/stable/2026.1/oslo_messaging/rpc/client.py)).
  The request may still have been processed: a timeout says the reply did not come back, not that nothing happened.
- **Usual causes:**
  - the broker is partitioned, blocked by a memory or disk alarm, or missing a node (then the timeouts appear in many
    services at once);
  - the server side (conductor, compute, agent) is down, overloaded, or stuck on a slow operation;
  - one specific call is slow by nature and exceeds the timeout.
- **Read next:** the log of the service that **should have answered**, at the same time (find it by the RPC topic in
  the traceback). If the timeouts are everywhere, the RabbitMQ log first.
- **Fix:** [RPC timeouts caused by RabbitMQ](../recovery/rabbitmq-partition-or-alarm-rpc-timeouts.md) when the broker
  is the cause.
- **Lab sample** (Neutron agents, while the control plane was starting after all controllers were rebooted; Neutron
  wraps the timeout in its own retry message). The lab's 90-second stop of every RabbitMQ node did not produce a
  `MessagingTimeout`: the calls waited while the clients reconnected.

  ```text
  2026-10-27 06:52:15.108 7 ERROR neutron_lib.rpc [None req-<request-id> - - - - - -] Timeout in RPC method has_alive_neutron_server. Waiting for 11 seconds before next attempt. If the server is not down, consider increasing the rpc_response_timeout option as Neutron server(s) may be overloaded and unable t...
  2026-10-27 06:52:15.108 7 WARNING neutron_lib.rpc [None req-<request-id> - - - - - -] Increasing timeout for has_alive_neutron_server calls to 120 seconds. Restart the agent to restore it to the default value.: oslo_messaging.exceptions.MessagingTimeout: Timed out waiting for a reply to message ID <msg-id>
  ```

### `[<connection-id>] AMQP server on <host>:<port> is unreachable: <error>. Trying again in <n> seconds.`

- **Where:** ERROR, in any service that uses RabbitMQ
  ([`impl_rabbit.py`](https://github.com/openstack/oslo.messaging/blob/stable/2026.1/oslo_messaging/_drivers/impl_rabbit.py)).
  The recovery is logged as `[<connection-id>] Reconnected to AMQP server on <host>:<port> via [<transport>] client
  with port <port>.`
- **Means:** the client cannot open a connection to that RabbitMQ node. It retries, and tries the other hosts of
  `transport_url`.
- **Usual causes:** the RabbitMQ node or its container is down, its port is filtered, or the name in `transport_url`
  does not resolve or points to the wrong address.
- **Read next:** on that node, the RabbitMQ log and the state of the container; then whether other services report the
  same node.
- **Fix:** [RPC timeouts caused by RabbitMQ](../recovery/rabbitmq-partition-or-alarm-rpc-timeouts.md) (missing node).
- **Lab sample** (`nova-compute`, RabbitMQ stopped on one controller, then on all three). The `<error>` was, in turn,
  the broker's shutdown notice, `RecoverableConnectionError`, a reset, then `ECONNREFUSED` while the port was closed;
  with one node down, `nova-compute` was reconnected to another node 10 s after its first error; with all three down
  they retried every second until a node was back. Requests in flight also log `Connection failed: <error> (retrying
  in <n> seconds)`.

  ```text
  2026-10-27 07:31:59.842 6 ERROR oslo.messaging._drivers.impl_rabbit [-] [<conn-id>] AMQP server on 192.0.2.13:5672 is unreachable: <RecoverableConnectionError: unknown error>. Trying again in 1 seconds.: amqp.exceptions.RecoverableConnectionError: <RecoverableConnectionError: unknown error>
  2026-10-27 07:32:00.845 6 ERROR oslo.messaging._drivers.impl_rabbit [-] [<conn-id>] AMQP server on 192.0.2.13:5672 is unreachable: [Errno 111] ECONNREFUSED. Trying again in 1 seconds.: ConnectionRefusedError: [Errno 111] ECONNREFUSED
  2026-10-26 21:08:22.136 21 ERROR oslo.messaging._drivers.impl_rabbit [-] [<conn-id>] AMQP server on 192.0.2.11:5672 is unreachable: (0, 0): (320) CONNECTION_FORCED - broker forced connection closure with reason 'shutdown'. ...
  2026-10-27 07:35:42.188 6 ERROR oslo.messaging._drivers.impl_rabbit [None req-<request-id> - - - - - -] Connection failed: [Errno 111] ECONNREFUSED (retrying in 0 seconds): ConnectionRefusedError: [Errno 111] ECONNREFUSED
  2026-10-27 07:37:17.874 6 INFO oslo.messaging._drivers.impl_rabbit [-] [<conn-id>] Reconnected to AMQP server on 192.0.2.12:5672 via [amqp] client with port 55436.
  ```

### `[<connection-id>] AMQP server <host>:<port> closed the connection. Check login credentials: <error>`

- **Where:** ERROR, same file, when the error contains `Socket closed`.
- **Means:** the TCP connection opened, then was closed. The message points at the credentials, but it is logged for
  **any** `Socket closed` error, including a broker shutting down or a proxy cutting the connection.
- **Usual causes:** wrong user or password in `transport_url`, a user or virtual host missing on the broker (for
  example after a broker rebuild); or a broker restart, or something in between closing connections.
- **Read next:** the RabbitMQ log on that node at the same second. A refused login shows there as `login refused` (see
  [reading the RabbitMQ logs](reading-rabbitmq-logs.md));
  without it, look for a broker stop or a connection closed by the network instead of a credentials problem.

### `A recoverable connection/channel error occurred, trying to reconnect: <error>`

- **Where:** INFO, from the client's heartbeat thread (same file).
- **Means:** the heartbeat found the connection broken (timeout, refused, reset) and the client reconnects.
- **Usual causes:** the broker restarted or dropped the connection (for example the broker's `missed heartbeats from
  client`), or the network between them broke.
- **Lab sample:** `... INFO oslo.messaging._drivers.impl_rabbit [-] A recoverable connection/channel error occurred,
  trying to reconnect: [Errno 104] Connection reset by peer` (a RabbitMQ node stopped).
- **Read next:** the RabbitMQ log for `missed heartbeats from client` or `closing AMQP connection` with the same
  client at the same time. One line is normal during a broker restart; many lines across services are not.

### `Unexpected error during heartbeat thread processing, retrying...`

- **Where:** WARNING, same thread (same file); the exception is only logged at DEBUG (`Exception` with the traceback).
- **Means:** the heartbeat thread hit an error it does not classify as a recoverable connection error.
- **Read next:** turn on debug for that service briefly to see the exception, or correlate with the broker log.

### Nova: `Timed out waiting for nova-conductor.  Is it running? Or did this service start before nova-conductor?  Reattempting establishment of nova-conductor connection...`

- **Where:** WARNING, at the start of `nova-compute` (and other services that wait for the conductor), from
  [`nova/conductor/api.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/conductor/api.py).
- **Means:** the service sent its first RPC call to `nova-conductor` and got `MessagingTimeout`. It keeps retrying;
  `nova-conductor connection established successfully` marks the end.
- **Read next:** is `nova-conductor` running and connected to the same broker and virtual host; the broker state.
- **Lab sample** (`nova-compute` started while the database was still down after all nodes were rebooted; 11
  minutes between the two lines):

  ```text
  2026-10-27 06:51:24.717 6 WARNING nova.conductor.api [None req-<request-id> - - - - - -] Timed out waiting for nova-conductor.  Is it running? Or did this service start before nova-conductor?  Reattempting establishment of nova-conductor connection...: oslo_messaging.exceptions.MessagingTimeout: ...
  2026-10-27 07:02:41.161 6 INFO nova.conductor.api [None req-<request-id> - - - - - -] nova-conductor connection established successfully
  ```

### Nova: `Lost connection to nova-conductor for reporting service status.`

- **Where:** WARNING, in `nova-compute`, from the database service group driver
  ([`servicegroup/drivers/db.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/servicegroup/drivers/db.py)).
  Logged once per outage; `Recovered from being unable to report status.` marks the end.
- **Means:** the periodic heartbeat that `nova-compute` sends through the conductor (it has no direct database
  access; the other Nova services write their heartbeat to the database themselves) timed out. If it lasts longer than `service_down_time`
  (60 s), the API shows this service `down` while it still runs.
- **On a lab** this line did not appear, neither with all RabbitMQ nodes stopped for 90 s nor with the database
  unreachable. With the conductor unable to reach the database, `nova-compute` logged instead
  `Unexpected error while reporting service status: oslo_messaging.rpc.client.RemoteError: Remote error:
  OperationalError (pymysql.err.OperationalError) (9001, ...)`, then, at INFO, `Recovered from being unable to report
  status.` The conductor and scheduler log the same pair for their own database heartbeat.
- **Read next:** the conductor's log and the broker state at that time.
- **Fix:** [RPC timeouts caused by RabbitMQ](../recovery/rabbitmq-partition-or-alarm-rpc-timeouts.md); for a compute
  shown down while its instances run, [nova-compute down, instances still running](../recovery/nova-compute-down-instances-still-running.md).

## Database (oslo.db with MariaDB/Galera)

### `DBConnectionError: (pymysql.err.OperationalError) (<code>, "<text>")`

- **Where:** in a traceback, in any service that talks to the database. oslo.db turns these MySQL errors into
  `DBConnectionError` (`@filters("mysql", sqla_exc.OperationalError, …2002|2003|2006|2013|1047…)` in
  [`exc_filters.py`](https://github.com/openstack/oslo.db/blob/stable/2026.1/oslo_db/sqlalchemy/exc_filters.py)).
  The exact text after the exception name depends on the driver; the code is what matters.
- **Lab sample** (ProxySQL stopped on every controller, so the internal VIP was gone; then a connection lost during
  a query):

  ```text
  2026-10-27 07:40:28.433 31 ERROR nova.servicegroup.drivers.db [-] Unexpected error while reporting service status: oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on '192.0.2.10' (timed out)")
  2026-10-27 07:41:44.754 7 WARNING oslo_db.sqlalchemy.engines [None req-<request-id> - - - - - -] SQL connection failed. -1 attempts left.: oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on '192.0.2.10' ([Errno 113] No route to host)")
  2026-10-27 07:14:01.143 23 CRITICAL keystone [None req-<request-id> - - - - - -] Unhandled error: oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2013, 'Lost connection to MySQL server during query')
  ```

  The text in brackets after 2003 varied with the state of the VIP: `timed out`, `[Errno 113] No route to host`,
  `[Errno 113] EHOSTUNREACH`, `[Errno 111] Connection refused`.
- **Behind ProxySQL, look for `OperationalError (9001, …)` too:** when ProxySQL runs but has no healthy backend for
  the query, the code is ProxySQL's own `9001` (`Max connect timeout reached while reaching hostgroup 0 after
  10000ms`). 9001 is **not** in oslo.db's `DBConnectionError` filter (2002, 2003, 2006, 2013, 1047), so this cause
  mostly shows as a plain `sqlalchemy.exc.OperationalError` or `pymysql.err.OperationalError (9001, …)`, not as
  `DBConnectionError`; on a lab it appeared twice as `DBConnectionError` and otherwise as `OperationalError`. Search
  for both.
- **Means, by code** (names from PyMySQL's
  [`CR.py`](https://github.com/PyMySQL/PyMySQL/blob/main/pymysql/constants/CR.py) and
  [`ER.py`](https://github.com/PyMySQL/PyMySQL/blob/main/pymysql/constants/ER.py)):

  | Code | Name | What it says |
  |---|---|---|
  | 2002 | `CR_CONNECTION_ERROR` | Cannot connect (local socket or name) |
  | 2003 | `CR_CONN_HOST_ERROR` | Cannot connect to the server host and port |
  | 2006 | `CR_SERVER_GONE_ERROR` | The server went away on an open connection |
  | 2013 | `CR_SERVER_LOST` | The connection was lost during a query |
  | 1047 | `UNKNOWN_COM_ERROR` | The server refused the command; with Galera, a node that is not synced (`WSREP has not yet prepared node for application use`) answers this |

- **Usual causes:** the database proxy has no healthy node (ProxySQL by default on Kolla-Ansible 2026.1, since
  `enable_proxysql` follows `enable_mariadb`, then usually with code 9001 as above; HAProxy's `mariadb` backend only
  when ProxySQL is disabled), the proxy itself or its VIP is gone (code 2003), the MariaDB
  container or service is down, Galera lost
  quorum or a node is not synced, or the connection was idle longer than the server or the proxy allows (2006/2013
  appearing once after a quiet period).
- **Read next:** the database proxy that is actually in the path: on Kolla-Ansible by default ProxySQL,
  `/var/log/kolla/proxysql/proxysql.log` on the load balancer nodes and the state of its Galera hostgroups (see
  [reading the MariaDB and Galera logs](reading-mariadb-galera-logs.md)); with ProxySQL disabled, HAProxy's `mariadb`
  backend ([reading the HAProxy, keepalived and memcached logs](reading-haproxy-keepalived-memcached-logs.md)). Then
  the MariaDB log on the nodes (Galera state: `wsrep_cluster_status`, `wsrep_local_state_comment`), as in
  [reading the MariaDB and Galera logs](reading-mariadb-galera-logs.md).
- **Fix:** [Galera lost quorum or stopped everywhere](../recovery/galera-cluster-lost-quorum-database-unavailable.md).

### `SQL connection failed. <n> attempts left.`

- **Where:** WARNING, when a service starts and tests its database connection
  ([`engines.py`](https://github.com/openstack/oslo.db/blob/stable/2026.1/oslo_db/sqlalchemy/engines.py)).
- **Means:** the first connection failed with `DBConnectionError`, and oslo.db retries every `retry_interval` (10 s by
  default) ([`options.py`](https://github.com/openstack/oslo.db/blob/stable/2026.1/oslo_db/options.py)). How it ends
  depends on `[database] max_retries`:
  - **oslo.db default, `max_retries = 10`:** after the last attempt, the service fails to start.
  - **Kolla-Ansible 2026.1, `max_retries = -1`** (set in the services' configuration templates): oslo.db retries
    forever (`itertools.count()` in `_test_connection`), so the service **never gives up**; it waits until the
    database answers. The warning logs `max_retries - attempt`, so the counter goes negative: on a lab, a
    `nova_scheduler` restarted while the database VIP was gone logged `SQL connection failed. -1 attempts left.` up to
    `-5 attempts left.` (07:41:44 to 07:42:33), then started normally once the database was back.
- **Read next:** same as `DBConnectionError`. With the oslo.db default, a service that failed to start can then loop
  under its unit; see [the systemd journal and the container engine](systemd-journal-and-container-engine.md).

### `DBDeadlock` (MySQL error 1213, `Deadlock found when trying to get lock`)

- **Where:** in a traceback; oslo.db maps error 1213 to `DBDeadlock` (`exc_filters.py`).
- **Means:** the database rolled back one transaction to resolve a lock conflict. With Galera, a certification
  conflict between nodes is reported to the client with the same error 1213, and the two look identical (see
  [reading the MariaDB and Galera logs](reading-mariadb-galera-logs.md)). oslo.db offers a retry on deadlock
  (`wrap_db_retry(retry_on_deadlock=True)` and the `retry_on_deadlock` decorator in
  [`oslo_db/api.py`](https://github.com/openstack/oslo.db/blob/stable/2026.1/oslo_db/api.py)) that the services use on
  many database calls, so an occasional line is usually harmless.
- **Read next:** only if they are frequent: whether services write through several Galera nodes at once, and the
  database load.

## A short reading order

1. Count the services and hosts that log these messages in the same minutes: everywhere means the broker or the
   database; one service means that service or its host.
2. Messaging: the first `is unreachable` or `MessagingTimeout`, then the RabbitMQ log at that second.
3. Database: the error code, then the database proxy in the path (ProxySQL by default on Kolla-Ansible, or HAProxy's
   `mariadb` backend), then Galera's state on each node.

## Validity and limits

- Written from upstream source: oslo.messaging, oslo.db and Nova `stable/2026.1`, PyMySQL (main branch) for the
  error code names. The messages in the headings and explanations are format strings from that source; only the *Lab
  sample* blocks are captured lines. The text that a database driver adds after the exception name varies.
- Lab samples: Kolla-Ansible 22.2.0 (2026.1), three controllers (RabbitMQ cluster, Galera behind ProxySQL), two
  computes, 2026-10-27: RabbitMQ stopped on one node for 2 minutes, on all three for 90 s during an instance boot (the
  instance went ACTIVE once RabbitMQ was back), ProxySQL stopped on all controllers for 2 minutes with `nova_scheduler`
  restarted meanwhile. Not observed on the lab: `closed the connection. Check login credentials`, `Unexpected error
  during heartbeat thread processing`, `Lost connection to nova-conductor for reporting service status.`, `DBDeadlock`,
  and the codes 2002, 2006 and 1047 in a service log. The `MessagingTimeout` and `Timed out waiting for nova-conductor`
  samples come from the lab's control plane starting after all its nodes were rebooted, not from an injection.
- The reading of 1047 as `WSREP has not yet prepared node for application use` is the line quoted in the Galera
  runbook; the codes and their names are from PyMySQL.
- Only the RabbitMQ driver of oslo.messaging is covered.

---

*Provenance: written by agent:ozarland-author, from upstream source code (oslo.messaging, oslo.db, Nova, PyMySQL) and
lab samples from Kolla-Ansible 2026.1 (2026-10-27, by the lab reviewer).
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs/error-messages-messaging-and-database.md&title=Feedback%3A+logs/error-messages-messaging-and-database.md):
your release, your deployment method, and what happened.
