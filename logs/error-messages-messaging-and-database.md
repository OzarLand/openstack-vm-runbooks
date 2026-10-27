---
title: "Error messages: messaging and database. MessagingTimeout, AMQP connection and heartbeat lines, DBConnectionError"
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [rabbitmq, mariadb, nova, neutron, cinder]
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
the upstream source at `stable/2026.1`, not captured lines; `<…>` marks a value.

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

### `[<connection-id>] AMQP server <host>:<port> closed the connection. Check login credentials: <error>`

- **Where:** ERROR, same file, when the error contains `Socket closed`.
- **Means:** the TCP connection opened, then the broker closed it. The message itself points at the credentials.
- **Usual causes:** wrong user or password in `transport_url`, a user or virtual host missing on the broker (for
  example after a broker rebuild).
- **Read next:** the RabbitMQ log on that node for the refused login, at the same second.

### `A recoverable connection/channel error occurred, trying to reconnect: <error>`

- **Where:** INFO, from the client's heartbeat thread (same file).
- **Means:** the heartbeat found the connection broken (timeout, refused, reset) and the client reconnects.
- **Usual causes:** the broker restarted or dropped the connection (for example the broker's `missed heartbeats from
  client`), or the network between them broke.
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

### Nova: `Lost connection to nova-conductor for reporting service status.`

- **Where:** WARNING, in `nova-compute` (and other Nova services), from the database service group driver
  ([`servicegroup/drivers/db.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/servicegroup/drivers/db.py)).
  Logged once per outage; `Recovered from being unable to report status.` marks the end.
- **Means:** the periodic heartbeat sent through the conductor timed out. If it lasts longer than `service_down_time`
  (60 s), the API shows this service `down` while it still runs.
- **Read next:** the conductor's log and the broker state at that time.
- **Fix:** [RPC timeouts caused by RabbitMQ](../recovery/rabbitmq-partition-or-alarm-rpc-timeouts.md); for a compute
  shown down while its instances run, [nova-compute down, instances still running](../recovery/nova-compute-down-instances-still-running.md).

## Database (oslo.db with MariaDB/Galera)

### `DBConnectionError: (pymysql.err.OperationalError) (<code>, "<text>")`

- **Where:** in a traceback, in any service that talks to the database. oslo.db turns these MySQL errors into
  `DBConnectionError` (`@filters("mysql", sqla_exc.OperationalError, …2002|2003|2006|2013|1047…)` in
  [`exc_filters.py`](https://github.com/openstack/oslo.db/blob/stable/2026.1/oslo_db/sqlalchemy/exc_filters.py)).
  The exact text after the exception name depends on the driver; the code is what matters.
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

- **Usual causes:** HAProxy has no healthy database backend, the MariaDB container or service is down, Galera lost
  quorum or a node is not synced, or the connection was idle longer than the server or the proxy allows (2006/2013
  appearing once after a quiet period).
- **Read next:** HAProxy's state for the database backend
  ([reading the HAProxy, keepalived and memcached logs](reading-haproxy-keepalived-memcached-logs.md)), then the
  MariaDB log on the nodes (Galera state: `wsrep_cluster_status`, `wsrep_local_state_comment`), as in
  [reading the MariaDB and Galera logs](reading-mariadb-galera-logs.md).
- **Fix:** [Galera lost quorum or stopped everywhere](../recovery/galera-cluster-lost-quorum-database-unavailable.md).

### `SQL connection failed. <n> attempts left.`

- **Where:** WARNING, when a service starts and tests its database connection
  ([`engines.py`](https://github.com/openstack/oslo.db/blob/stable/2026.1/oslo_db/sqlalchemy/engines.py)).
- **Means:** the first connection failed with `DBConnectionError`; oslo.db retries up to `max_retries` (10 by
  default) every `retry_interval` (10 s by default)
  ([`options.py`](https://github.com/openstack/oslo.db/blob/stable/2026.1/oslo_db/options.py)). After the last
  attempt, the service fails to start and, on Kolla-Ansible, its unit restarts it.
- **Read next:** same as `DBConnectionError`; on a service in a restart loop, see the systemd and container engine
  page.

### `DBDeadlock` (MySQL error 1213, `Deadlock found when trying to get lock`)

- **Where:** in a traceback; oslo.db maps error 1213 to `DBDeadlock` (`exc_filters.py`).
- **Means:** the database rolled back one transaction to resolve a lock conflict. oslo.db offers a retry on deadlock
  (`wrap_db_retry(retry_on_deadlock=True)` and the `retry_on_deadlock` decorator in
  [`oslo_db/api.py`](https://github.com/openstack/oslo.db/blob/stable/2026.1/oslo_db/api.py)) that the services use on
  many database calls, so an occasional line is usually harmless.
- **Read next:** only if they are frequent: whether services write through several Galera nodes at once, and the
  database load.

## A short reading order

1. Count the services and hosts that log these messages in the same minutes: everywhere means the broker or the
   database; one service means that service or its host.
2. Messaging: the first `is unreachable` or `MessagingTimeout`, then the RabbitMQ log at that second.
3. Database: the error code, then HAProxy's backend state, then Galera's state on each node.

## Validity and limits

- Written from upstream source only: oslo.messaging, oslo.db and Nova `stable/2026.1`, PyMySQL (main branch) for the
  error code names. Every message is a format string from that source, not a captured line; the text that a database
  driver adds after the exception name varies.
- Not run on a lab yet. A lab capture of RabbitMQ stopped on one node, then on all nodes, and of the database backend
  made unavailable behind HAProxy, is requested to add real redacted lines.
- The reading of 1047 as `WSREP has not yet prepared node for application use` is the line quoted in the Galera
  runbook; the codes and their names are from PyMySQL.
- Only the RabbitMQ driver of oslo.messaging is covered.

---

*Provenance: written by agent:ozarland-author, from upstream source code (oslo.messaging, oslo.db, Nova, PyMySQL).
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*
