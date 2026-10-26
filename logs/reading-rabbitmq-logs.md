---
title: Reading the RabbitMQ logs, and the oslo.messaging lines that go with them
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [rabbitmq, nova, neutron, cinder]
deployment_methods: [kolla-ansible, any]
environment:
  topology: multinode, 3 RabbitMQ nodes
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by: []
upstream_refs:
  - https://www.rabbitmq.com/docs/4.2/partitions
  - https://www.rabbitmq.com/docs/4.2/alarms
  - https://www.rabbitmq.com/docs/4.2/memory
  - https://www.rabbitmq.com/docs/4.2/disk-alarms
  - https://www.rabbitmq.com/docs/4.2/flow-control
  - https://www.rabbitmq.com/docs/4.2/heartbeats
  - https://www.rabbitmq.com/docs/4.2/metadata-store
  - https://github.com/rabbitmq/rabbitmq-server/tree/v4.2.9/deps/rabbit/src
  - https://opendev.org/openstack/oslo.messaging/src/branch/stable/2026.1/oslo_messaging/_drivers/impl_rabbit.py
  - https://opendev.org/openstack/oslo.messaging/src/branch/stable/2026.1/oslo_messaging/_drivers/amqpdriver.py
  - https://opendev.org/openstack/kolla-ansible/src/branch/stable/2026.1/ansible/roles/rabbitmq/templates/rabbitmq.conf.j2
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Reading the RabbitMQ logs, and the oslo.messaging lines that go with them

When RabbitMQ misbehaves, the symptoms appear in every OpenStack service at once, as `MessagingTimeout` and
reconnection lines, and the cause is in RabbitMQ's own log on one of its nodes. This page reads both sides: the
broker's log (node down, partition, alarms, connections) and the client lines that `oslo.messaging` writes in the
OpenStack services' logs. Broker messages are quoted from the RabbitMQ 4.2.9 source (`deps/rabbit/src/`), the
version line Kolla 2026.1 installs (`rabbitmq-server-4.2.*` in its `rabbitmq` image); client messages from
`oslo.messaging` 2026.1.

The fix for a broken RabbitMQ cluster is not on this page; it says what the lines mean and where to look next.

## Where the broker log is, and its format

**Kolla-Ansible:** `RABBITMQ_LOG_BASE=/var/log/kolla/rabbitmq` (`rabbitmq-env.conf.j2`), and the node's log is
`/var/log/kolla/rabbitmq/rabbit@<hostname>.log`, on each RabbitMQ node (the controllers by default).

**Any:** the log location depends on the package; `rabbitmq-diagnostics log_location` prints it on a running node.

A line starts with a timestamp, the level in brackets, and the Erlang process id of the process that logged:

```text
<date> <time> [<level>] <0.N.0> <message>
```

RabbitMQ's own guides show lines in that shape, for example (from the
[memory guide](https://www.rabbitmq.com/docs/4.2/memory)):

```text
2023-06-10 23:17:05.976 [info] <0.308.0> Memory high watermark set to 1024 MiB (1073741824 bytes) of 8192 MiB (8589934592 bytes) total
```

Kolla-Ansible's fluentd parser accepts the timestamp with or without microseconds and a `+HH:MM` offset
(`input/03-rabbitmq.conf.j2`). Several messages span several lines (a banner of asterisks, a table); read the
lines after the first.

```bash
# Kolla-Ansible, on a RabbitMQ node
grep -nE '\[(warning|error)\]' /var/log/kolla/rabbitmq/rabbit@$(hostname).log | tail -30
docker exec rabbitmq rabbitmq-diagnostics cluster_status
docker exec rabbitmq rabbitmq-diagnostics alarms
```

## First: which metadata store, which partition handling

RabbitMQ 4.2 has two metadata stores. **Khepri** (Raft-based) is the default for new clusters from 4.2.0; a cluster
upgraded from earlier versions keeps **Mnesia** until the `khepri_db` feature flag is enabled
([metadata store guide](https://www.rabbitmq.com/docs/4.2/metadata-store)). Kolla-Ansible enables all feature
flags after an upgrade (`rabbitmqctl enable_feature_flag all` in `tasks/feature-flags.yml`) and writes
`cluster_partition_handling = pause_minority` (`rabbitmq_cluster_partition_handling` default).

What a partition looks like in the log depends on the store, and the node says which one at start
(`rabbit_node_monitor.erl`):

| Line at start | Meaning |
|---|---|
| `Starting rabbit_node_monitor (partition handling strategy unapplicable with Khepri)` | Khepri: `cluster_partition_handling` has no effect; availability follows Raft majorities. |
| `Starting rabbit_node_monitor (in pause_minority mode)` | Mnesia, with Kolla-Ansible's setting. |
| `Starting rabbit_node_monitor (in <mode> mode), might be partitioned from [<nodes>]` | Mnesia, and the node already sees peers it may be partitioned from at start. |

```bash
grep -h "Starting rabbit_node_monitor" /var/log/kolla/rabbitmq/rabbit@*.log | tail -1
docker exec rabbitmq rabbitmqctl list_feature_flags name state | grep khepri_db
```

OpenStack services on Kolla-Ansible 2026.1 use quorum queues (`rabbit_quorum_queue = true` in the service
templates), which are Raft-based whatever the metadata store: a quorum queue keeps working while a majority of its
replicas is reachable.

## Nodes going down and partitions

| Line (message text) | Level | What it means |
|---|---|---|
| `node <node> down: <reason>` | info | The Erlang distribution connection to that node was lost. With `net_tick_timeout` as the reason, the peer was silent for the [net tick time](https://www.rabbitmq.com/docs/4.2/nettick) (60 s by default); the net tick guide shows the line `node rabbit@peer-hostname down: net_tick_timeout`. |
| `rabbit on node <node> down` | info | RabbitMQ (the application) on that node is gone from the cluster's view. |
| `node <node> up` / `rabbit on node <node> up` | info | It is back. |
| `Node <node> was restarted` | warning | It came back with a new identity, so it restarted rather than reconnected. |
| `Partial partition detected:` / ` * We saw DOWN from <node>` / ` * We can still see <node> which can see <node>` then, with `pause_minority`, ` * pause_minority mode enabled` / `We will therefore pause until the *entire* cluster recovers` | error | Mnesia: this node lost one peer while another peer can still see it. RabbitMQ turns a partial partition into a full one: it pauses (pause_minority) or disconnects from the proxy node (`We will therefore intentionally disconnect from <node>`). |
| `Partial partition disconnect from <node>` | error | Mnesia: the other side asked this node to disconnect, to complete the partition. |
| `Cluster minority/secondary status detected - awaiting recovery` | warning | Mnesia, `pause_minority`: this node is in a minority and **stops RabbitMQ** until it sees the rest of the cluster again. Its listeners close; clients connected to it reconnect elsewhere. |
| `Mnesia(<node>): ** ERROR ** mnesia_event got {inconsistent_database, running_partitioned_network, <node>}` | error | Mnesia: two nodes that each thought the other was down are back in contact. A partition happened. RabbitMQ's [partitions guide](https://www.rabbitmq.com/docs/4.2/partitions) shows this line. |
| `Autoheal: we were selected to restart; winner is <node>` and other `Autoheal …` lines | warning, info | Mnesia, only with `cluster_partition_handling = autoheal` (not Kolla-Ansible's default). |

Sources: `rabbit_node_monitor.erl`, `rabbit_autoheal.erl` (RabbitMQ 4.2.9); the partitions guide.

To confirm, ask the nodes rather than the log:

```bash
docker exec rabbitmq rabbitmq-diagnostics cluster_status     # "Network Partitions" section, running nodes
docker exec rabbitmq rabbitmq-diagnostics check_if_node_is_quorum_critical
```

## Resource alarms: publishers blocked

A node raises an alarm when its memory use crosses the high watermark (default `0.6` of detected RAM), or its free
disk falls below `disk_free_limit` (default 50 MB). **The alarm is cluster-wide**: every node blocks publishing
connections until it clears ([alarms](https://www.rabbitmq.com/docs/4.2/alarms),
[memory](https://www.rabbitmq.com/docs/4.2/memory), [disk](https://www.rabbitmq.com/docs/4.2/disk-alarms)). For
OpenStack, this means every RPC call stops: services time out, nothing errors in RabbitMQ beyond the alarm.

| Line (message text) | Level | What it means |
|---|---|---|
| `<source> resource limit alarm set on node <node>.` followed by `*** Publishers will be blocked until this alarm clears ***` in a banner | warning | `<source>` is `memory` or `disk`. From now on, the cluster blocks publishers. |
| `<source> resource limit alarm cleared on node <node>` | warning | That node's alarm cleared. |
| `<source> resource limit alarm cleared across the cluster` | warning | No node has this alarm any more; publishing resumes. |
| `file descriptor limit alarm set.` with `*** New connections will not be accepted until this alarm clears ***` | warning | The node ran out of file descriptors: no new client connection. |
| `Memory high watermark set to <n> MiB (<n> bytes) of <n> MiB (<n> bytes) total` | info | At start: the limit this node computed. Check it against the memory the container or host really has. |
| `Disk free limit set to <n>MB` | info | At start: the disk limit (the disk guide shows this line). |

Sources: `rabbit_alarm.erl`, `vm_memory_monitor.erl` (RabbitMQ 4.2.9); the alarm, memory and disk guides.

On the client side, `oslo.messaging` logs the broker's block and unblock:

| Line (message text) | Level | Log |
|---|---|---|
| `The broker has blocked the connection: <reason>` | ERROR | every OpenStack service connected to the cluster |
| `The broker has unblocked the connection` | INFO | same |

Source: `impl_rabbit.py`. Seeing the first in many services at the same second points at an alarm; read the broker
log for which node and which resource.

**Flow control** is different: a connection in state `flow` is being slowed many times a second because queues
cannot keep up. It is normal under load, logs nothing, and shows only in `rabbitmqctl list_connections name state`
or the management UI ([flow control](https://www.rabbitmq.com/docs/4.2/flow-control)). A connection stuck in
`blocked` or `blocking` is the alarm above, not flow control.

## Client connections: churn and heartbeats

Each OpenStack service process opens AMQP connections; each connection logs on the broker when it opens and
closes (`rabbit_reader.erl`):

| Line (message text) | Level | What it means |
|---|---|---|
| `accepting AMQP connection <connection name>` | info | A client connected. A TCP connection that sends no data (a load balancer health check) is logged at debug only. |
| `connection <name> has a client-provided name: <name>` | info | `oslo.messaging` names its connections; the name shows which process (and host) it is. |
| `closing AMQP connection (<client>, vhost: '<vhost>', user: '<user>', duration: '<duration>')` | info | A clean close: a service stopped or restarted. |
| `closing AMQP connection <pid> (<client>, vhost: '<vhost>', user: '<user>', duration: '<duration>'):` / `client unexpectedly closed TCP connection` | warning | The client went away without closing: its process died, or a network device (HAProxy, firewall) cut an idle connection. |
| `closing AMQP connection <pid> (<client>, duration: '<duration>'):` / `missed heartbeats from client, timeout: <n>s` | error | The broker heard nothing from the client for the negotiated heartbeat timeout. The client process is stuck (blocked event loop, CPU starvation), or the network between them dropped. |
| `<mechanism> login refused: user '<user>' - invalid credentials` | error | A service uses a wrong password, typically after a password change not deployed everywhere. |

**Churn** (many accept and close lines a minute for the same clients) means services reconnect in a loop. Count
them per client to find which host or service:

```bash
grep -c "accepting AMQP connection" /var/log/kolla/rabbitmq/rabbit@$(hostname).log
grep "missed heartbeats from client" /var/log/kolla/rabbitmq/rabbit@$(hostname).log | tail
```

On the client side, `oslo.messaging` logs (`impl_rabbit.py`; prefix `[<connection-id>]`):

| Line (message text) | Level | What it means |
|---|---|---|
| `[<id>] AMQP server on <host>:<port> is unreachable: <error>. Trying again in <n> seconds.` | ERROR | The service cannot connect to that RabbitMQ node; it retries (and tries the other nodes of `transport_url`). |
| `[<id>] AMQP server <host>:<port> closed the connection. Check login credentials: <error>` | ERROR | The TCP connection succeeded but RabbitMQ closed it (`Socket closed`): usually refused credentials or vhost. |
| `[<id>] Reconnected to AMQP server on <host>:<port> via [<transport>] client with port <port>.` | INFO | Reconnected. A burst of these across services marks the end of a broker or network incident. |
| `A recoverable connection/channel error occurred, trying to reconnect: <error>` | INFO | The heartbeat thread found the connection broken and reconnects. |
| `Failed to publish message to topic '<topic>': <error>` | ERROR | A send failed after the retries. |
| `Timed out waiting for a reply to message ID <msg-id>` (in a `MessagingTimeout`) | ERROR, in a traceback | An RPC call got no answer within `rpc_response_timeout` (60 s). Either the server side (conductor, agent, compute) did not process it, or the reply was lost. (`amqpdriver.py`) |

The client heartbeat is `heartbeat_timeout_threshold` (default 60 s), checked `heartbeat_rate` times (default 3) in
that window (`[oslo_messaging_rabbit]`).

## Reading a RabbitMQ incident in order

1. **Is it everywhere?** `MessagingTimeout` or `The broker has blocked the connection` in several services at the
   same minute points at RabbitMQ, not at those services.
2. **Alarms first:** `rabbitmq-diagnostics alarms` and `resource limit alarm set` in each node's log. An alarm on one
   node blocks the whole cluster.
3. **Membership:** `cluster_status`, then `node <node> down`, `Partial partition detected` and
   `Cluster minority/secondary status detected` lines, with their times, on each node.
4. **Connections:** churn and `missed heartbeats from client` point at clients or the network between them and the
   broker; group by client address to find the host.
5. **Then the clients:** the first `AMQP server … is unreachable` and the last `Reconnected to AMQP server` in a
   service bracket the incident as that service saw it.

How the shared dependencies fail together: [VM creation fails platform-wide](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md).

## Validity and limits

- Broker messages are quoted from the RabbitMQ **4.2.9** source and the RabbitMQ 4.2 guides; client messages from
  `oslo.messaging` 2026.1. **No line on this page was captured on a lab**, apart from the guide's own examples,
  which come from older RabbitMQ versions and show the shape, not 4.2 output. A lab request asks for real samples.
- Whether a Kolla-Ansible 2026.1 cluster runs Khepri or Mnesia depends on its history (new or upgraded) and on the
  feature flags; check with the commands above. The partition lines marked "Mnesia" do not appear with Khepri,
  whose partition handling relies on Raft and logs leader elections instead (not detailed here).
- Default values (watermark 0.6, disk 50 MB, net tick 60 s, heartbeat 60 s) are upstream defaults; your deployment
  may override them in `rabbitmq.conf` or `[oslo_messaging_rabbit]`.

## Related

- [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md)
- [VM creation fails platform-wide: check the shared dependencies first](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md)
- [Reopening provisioning after a control-plane incident](../recovery/reopening-provisioning-after-control-plane-incident.md)
- [Logs index](README.md)

---

*Provenance: written by agent:ozarland-author, from RabbitMQ 4.2 and oslo.messaging 2026.1 source code and
documentation. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Freading-rabbitmq-logs.md&title=Feedback%3A+logs%2Freading-rabbitmq-logs.md):
your release, your deployment method, and what happened.
