---
title: RPC timeouts in every service, computes reported down. RabbitMQ is partitioned, blocked by an alarm, or a node is missing
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [rabbitmq, oslo.messaging, kolla-ansible, nova, neutron, cinder]
deployment_methods: [kolla-ansible, any]
environment:
  topology: HA control plane, RabbitMQ cluster of three or more nodes
  network: any
  storage: any
  os: any
risk: destructive
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/message-queues/rabbitmq.html
  - https://www.rabbitmq.com/docs/4.2/partitions
  - https://www.rabbitmq.com/docs/4.2/alarms
  - https://www.rabbitmq.com/docs/4.2/metadata-store
  - https://www.rabbitmq.com/docs/4.2/quorum-queues
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# RPC timeouts in every service, computes reported down. RabbitMQ is partitioned, blocked by an alarm, or a node is missing

## Symptom

Several services at once log RPC timeouts:

```text
oslo_messaging.exceptions.MessagingTimeout: Timed out waiting for a reply to message ID <id>
```

and some of these follow:

- new instances stay in `BUILD` (scheduling), volumes stay in `creating`, actions on instances never finish;
- computes go `down` in `openstack compute service list` although `nova-compute` runs, and its log says
  `Lost connection to nova-conductor for reporting service status.`;
- Neutron agents flap between alive and dead;
- the APIs answer, slowly, because reads go to the database; actions that need a service behind the bus fail.

## Impact

The whole control plane, for every action that goes through the message bus: create, delete, migrate, attach.
Running instances keep running and keep their network (see
[control plane down](../diagnostics/control-plane-down-what-still-works-for-instances.md)). Requests sent while the
bus was broken may be lost or delivered twice, which leaves instances, volumes or ports in transitional states to
clean up afterwards.

## Quick checks

Read-only. On **every** RabbitMQ node.

**Kolla-Ansible:** prefix each command with `sudo docker exec rabbitmq`.

```bash
# 1. Which nodes this node sees, partitions, alarms
rabbitmq-diagnostics cluster_status

# 2. Resource alarms on this node (memory or disk)
rabbitmq-diagnostics check_local_alarms

# 3. Queues with messages waiting, and their consumers
rabbitmqctl list_queues name type messages consumers --silent | sort -k3 -n -r | head -20

# 4. Metadata store: is Khepri in use? (enabled = Khepri, otherwise Mnesia)
rabbitmqctl list_feature_flags name state --silent | grep khepri
```

On a controller, the container state and log:

```bash
sudo docker ps -a --filter name=rabbitmq --format '{{.Names}} {{.Status}}'
sudo ls /var/log/kolla/rabbitmq/                       # the node log is rabbit@<hostname>.log
sudo tail -n 100 /var/log/kolla/rabbitmq/rabbit@<hostname>.log
```

**Any:** the same commands without the prefix, on each RabbitMQ node, and the RabbitMQ log.

| What you see | Case |
|---|---|
| Every node lists all others as running, no partition, no alarm, no queue growing | RabbitMQ is healthy: look at the services and the database ([shared dependencies](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md)) |
| One node stopped or missing from the running nodes, the others agree | Cause 1: a node is down |
| *Network Partitions* lists nodes that cannot communicate | Cause 2: a network partition |
| An alarm in `cluster_status` or `check_local_alarms` | Cause 3: a memory or disk alarm |
| RPC queues (`conductor`, `scheduler`, `compute.<host>`, `q-plugin`…) growing, with 0 consumers | Cause 4: consumers gone |

## Diagnosis

### What Kolla-Ansible 2026.1 deploys

- RabbitMQ **4.2** (the 2026.1 release notes), with `cluster_partition_handling = pause_minority`
  (`rabbitmq_cluster_partition_handling`).
- OpenStack services use **quorum queues** (`rabbit_quorum_queue = true`) and **streams** for fanout
  (`rabbit_stream_fanout = true`, `om_enable_rabbitmq_stream_fanout`), with `use_queue_manager = true`
  (`nova.conf.j2` and the other service templates).
- Since RabbitMQ 4.2, Khepri is the default metadata store for a new cluster; a cluster upgraded from an older
  RabbitMQ keeps Mnesia until Khepri is enabled (RabbitMQ metadata store guide, 4.2).

### What each failure does

- **Quorum queues and Khepri** need a majority of their members. On the majority side, a new leader is elected and
  the queues keep working; on the minority side, they make no progress (RabbitMQ partitions guide). With three
  nodes, one node lost is absorbed; two lost stops the queues.
- **`pause_minority`** (relevant with Mnesia): nodes that are not in a strict majority pause, stop listening, check
  every second, and resume when they see the majority again. Clients connected to them reconnect elsewhere. The guide
  warns that it does not protect against a node that was suspended (a frozen VM, a long pause) and then resumed.
- **Alarms:** when a node's memory goes above its watermark, or its free disk below its limit, RabbitMQ blocks every
  publishing connection **on every node** of the cluster until the alarm clears (RabbitMQ alarms guide). Consumers
  keep working; publishers, that is every RPC caller, hang.
- **Streams** for fanout keep messages until the retention policy deletes them (Kolla-Ansible's defaults: 500 MB
  segments, kept 1800 s once full). Many ready messages in a `stream` queue are expected, not a backlog
  (Kolla-Ansible RabbitMQ guide).
- **nova-compute** reports its state through `nova-conductor` over RPC. When the bus fails, its reports stop, and
  Nova marks it `down` after `service_down_time` (60 s by default), while the compute and its instances are fine.
  `nova/servicegroup/drivers/db.py` logs `Lost connection to nova-conductor for reporting service status.` once when
  the reports start failing, and `Recovered from being unable to report status.` when they work again: the second
  line on every compute is a good sign that the bus is back.

## Possible causes

### Cause 1: one node down

A controller rebooted, the container crashed, the host ran out of memory. With three or more nodes the cluster keeps
working; clients connected to that node reconnect. Timeouts are short-lived.

### Cause 2: a network partition

The controllers lost contact with each other. `cluster_status` lists the partition. On the minority side, the nodes
pause (Mnesia with `pause_minority`) or their queue replicas stop making progress (quorum queues, Khepri).

### Cause 3: a memory or disk alarm

A node's disk filled up (often its logs, or the volume behind `/var/lib/rabbitmq`), or a backlog of messages used
its memory. One node's alarm blocks publishers on the whole cluster.

### Cause 4: consumers gone

The service that consumes a queue stopped, or cannot connect: every RPC to it waits until the timeout, and its queue
grows. The bus itself is fine; the problem is the service (container stopped, crash loop, database errors).

## Resolution

Fix the cause first, then check that the services recovered (see *Verification*): oslo.messaging reconnects on its
own once RabbitMQ is reachable again.

### For cause 1: start the node again

> ⚠️ **State-changing.** Starts RabbitMQ on that node; it rejoins the cluster, and its quorum queue replicas catch
> up from the leaders. Undo: stop the container.

```bash
sudo docker start rabbitmq
sudo docker exec rabbitmq rabbitmq-diagnostics cluster_status      # the node is back in the running nodes
```

Streams do not get a replica back on a node that was down when they were declared. Once the cluster is healthy,
Kolla-Ansible's guide points to `contrib/ops/rabbitmq/rabbitmq-repair-stream-replicas.sh` to add the missing
replicas, and warns that stream membership changes are not entirely safe: run it only on a healthy cluster.

### For cause 2: repair the network, then let the cluster heal

With quorum queues, Khepri, or `pause_minority`, the minority side catches up or resumes on its own once the network
is back. Check `cluster_status` on every node.

If a partition is still listed after the network is fixed (Mnesia, or a node that was suspended), the RabbitMQ
partitions guide says to choose the side you trust, restart the nodes of the other side, then restart the trusted
side to clear the warning.

> 🛑 **Destructive.** Changes made on the restarted side during the partition (queues, bindings, messages there) are
> lost: those nodes take their state from the trusted side. Clients on those nodes are disconnected and reconnect.
> Precondition: the network between all RabbitMQ nodes works again; you know which side the services used (the
> one where their queues have consumers). Restart one node at a time.

```bash
# on each node of the side you do not trust, one at a time
sudo docker restart rabbitmq
# then on each node of the trusted side, one at a time
sudo docker restart rabbitmq
```

### For cause 3: free the resource

Free disk space or memory on the node that raised the alarm: remove what filled the disk (see
[compute disk full](../lifecycle/compute-disk-full-instance-paused.md) for the method, the same applies to a
controller), or find the queue whose backlog grew (quick check 3) and fix its consumer (cause 4). The alarm clears
by itself when usage falls below the limit, and publishers are unblocked.

### For cause 4: bring the consumer back

Find the service named by the queue (`conductor` for `nova-conductor`, `scheduler`, `compute.<host>` for that
compute, `cinder-volume…`, `q-plugin` and the agent queues for Neutron) and fix it, from its own log. A restarted
service drains its queue.

> ⚠️ **State-changing.** Restarts one service container; its in-flight requests fail and are not retried. Undo: none
> needed.

```bash
sudo docker restart <service-container>          # for example nova_conductor
```

### Last resort: reset RabbitMQ's state

Kolla-Ansible's `rabbitmq-reset-state` stops the RabbitMQ application on each node, resets it (`force_reset`, or
`reset` with Khepri), starts it and waits for every node to join. The RabbitMQ guide of Kolla-Ansible uses it with
every OpenStack service stopped first, then redeployed.

> 🛑 **Destructive.** Every queue, exchange and message of the cluster is deleted; the services recreate their
> queues when they start. Requests in flight are lost, and the resources they concerned stay in transitional states.
> Every service is down from the stop to the deploy. Precondition: causes 1 to 4 ruled out; a maintenance window.
> Cannot be undone.

```bash
kolla-ansible genconfig -i <inventory>
kolla-ansible stop -i <inventory> --tags <service-tags> --yes-i-really-really-mean-it
kolla-ansible rabbitmq-reset-state -i <inventory>
kolla-ansible deploy -i <inventory> --tags <service-tags>
```

`<service-tags>`: the services that use RabbitMQ, for example `nova,neutron,cinder,glance,heat`.

## Verification

```bash
# on every node: all nodes running, no partition, no alarm
sudo docker exec rabbitmq rabbitmq-diagnostics cluster_status
# no RPC queue growing, and each one has consumers
sudo docker exec rabbitmq rabbitmqctl list_queues name type messages consumers --silent | sort -k3 -n -r | head
# from a host with admin credentials
openstack compute service list          # every compute up again within about a minute
openstack network agent list            # every agent alive
openstack volume service list
```

Then a test build, and the clean-up of what stayed stuck:
[reopening provisioning after a control-plane incident](reopening-provisioning-after-control-plane-incident.md) and
[stale records](../lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md).

## Validity and limits

- Written from Kolla-Ansible 2026.1 (RabbitMQ guide, `rabbitmq.conf.j2`, `reset-state.yml`, role defaults, release
  notes), the Nova 2026.1 source (`servicegroup/drivers/db.py`, `service_down_time`) and the RabbitMQ 4.2 partitions,
  alarms, metadata store and quorum queue guides. Not run on a lab.
- RabbitMQ 4.3 removes Mnesia and partition handling strategies; on a later Kolla-Ansible release, read that
  release's RabbitMQ version first.
- Not covered: TLS problems between services and RabbitMQ, credentials (see the password rotation pattern of
  kolla-ansible-deployment-patterns), RabbitMQ upgrades, and notification queues filled by a missing consumer
  (Ceilometer, Designate).

## Related

- [VM creation fails platform-wide. Check the shared dependencies first](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md)
- [Reopening provisioning after a control-plane incident](reopening-provisioning-after-control-plane-incident.md)
- [Volume stuck in creating](../volumes/volume-stuck-in-creating-or-error-after-create.md): a request lost on the bus
- Kolla-Ansible 2026.1: [RabbitMQ](https://docs.openstack.org/kolla-ansible/2026.1/reference/message-queues/rabbitmq.html)
- RabbitMQ 4.2: [partitions](https://www.rabbitmq.com/docs/4.2/partitions), [alarms](https://www.rabbitmq.com/docs/4.2/alarms),
  [metadata store](https://www.rabbitmq.com/docs/4.2/metadata-store), [quorum queues](https://www.rabbitmq.com/docs/4.2/quorum-queues)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible and Nova 2026.1 documentation and source and
the RabbitMQ 4.2 documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=recovery/rabbitmq-partition-or-alarm-rpc-timeouts.md&title=Feedback%3A+recovery/rabbitmq-partition-or-alarm-rpc-timeouts.md):
your release, your deployment method, and what happened.
