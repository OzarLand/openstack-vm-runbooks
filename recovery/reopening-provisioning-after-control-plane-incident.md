---
title: Reopening provisioning after a control-plane incident. Stop the APIs, recover, verify, reopen
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [nova, neutron, rabbitmq, mariadb, haproxy]
deployment_methods: [any]
environment:
  topology: HA control plane (clustered RabbitMQ, Galera)
  network: any
  storage: any
  os: any
risk: destructive
confirmed_by: []
upstream_refs:
  - https://www.rabbitmq.com/docs/partitions
  - https://galeracluster.com/library/documentation/crash-recovery.html
  - https://docs.openstack.org/kolla-ansible/latest/user/operating-kolla.html
authored_by: agent:ozarland-author
last_verified: 2026-10-18
---

# Reopening provisioning after a control-plane incident. Stop the APIs, recover, verify, reopen

Controlled degradation applied to a real incident: users could not create VMs, running instances were fine, the
message bus and the database were unhealthy, and the cause was unknown at the time. The team chose to stop
accepting requests, recover the shared infrastructure on a platform whose state was no longer changing, verify end
to end, and only then reopen. The alternative, repairing the bus and the database while the APIs kept accepting
operations, would have produced broader and less predictable damage.

## Symptom

See [VM creation fails platform-wide](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md):
provisioning lost, workloads running, RabbitMQ unable to complete RPC exchanges reliably (partition, alarms, or
queues with no consumers), Galera affected (nodes out of `Primary`, flow control stalled).

## Impact

Deliberate and explicit: provisioning and resource changes unavailable for the duration of the recovery. Running
instances keep running, keep their storage and keep exchanging traffic. New ports, security-group rule changes and
floating IP associations will not take effect until the control plane returns; an instance that boots during the
stop may fail to reach the metadata service.

## Quick checks: decide whether to stop

Stop the APIs when **both** are true:

1. The shared infrastructure is unhealthy and you do not yet know why or which requests were partially applied.
2. The business impact of "no new VMs for N minutes" is bounded and can be communicated.

Do **not** use this runbook for a single RabbitMQ node that left a cluster that still has a majority; rejoin that
node instead. The two situations start from the same alert and need different procedures.

Before stopping, record who authorised the stop and communicate it as a protection decision: what is restricted,
why continuing would create greater risk, what remains protected, and what condition will allow reopening.

## Diagnosis

Establish the state of the shared dependencies before deciding, read-only, with
[VM creation fails platform-wide: check the shared dependencies first](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md):
Galera status on every node, RabbitMQ cluster status, partitions, alarms and queues without consumers, compute
services and network agents reporting, and what changed recently. Write down what you observe before acting: it is
the evidence the incident review will need, and the state will change once you start recovering.

## Possible causes

| Cause | How to tell | Is this runbook the right one? |
|---|---|---|
| RabbitMQ network partition | `rabbitmqctl cluster_status` lists partitions (Mnesia); with Khepri, *Network Partitions* stays `(none)` and the nodes disagree on the running nodes | Yes. This was the cause in the source incident |
| RabbitMQ memory or disk alarm | `rabbitmq-diagnostics check_alarms` | Usually not: free memory or disk and the alarm clears; no stop needed |
| One RabbitMQ node left a cluster that keeps its majority | One node missing, no partition | No: rejoin or reset that node only |
| Galera lost quorum on one node | That node `non-Primary`, others `Primary` | No: restart that node and let it rejoin |
| Galera lost quorum on every node | All nodes `non-Primary` | Yes, combined with step 3 |
| Expired certificates or Fernet keys | Authentication fails everywhere at a precise time | No: see the certificates page in `diagnostics/` |

## Resolution

### Step 1: stop accepting requests

> ⚠️ **State-changing, production impact.** Every API becomes unavailable to users. Running instances are not
> affected. Stopping the APIs does not silence the platform: conductors, schedulers and agents keep their own work
> going.

Preferred: disable the API backends at the load balancer, so that the VIP answers with an error rather than
timing out, and the action is reversible in one place. Alternative: stop the API services or containers.

**Kolla-Ansible:** on each controller, stop the systemd units, not the containers: each container runs from a unit
with `Restart=always`, so a `docker stop` is undone within about a minute. For example
`systemctl stop kolla-nova_api-container.service kolla-neutron_server-container.service kolla-cinder_api-container.service kolla-glance_api-container.service`
(adapt to the services you run: a unit that does not exist on a node only prints `Failed to stop ...: Unit ... not
loaded.`, and Glance with a file backend runs on one controller); or set those backends to `MAINT` in HAProxy.
Keep `placement_api` running: on 2026.1, `nova-conductor` and `nova-scheduler` exit at start-up when the Placement
endpoint answers without a usable version (on the lab, HAProxy answering for the stopped backend:
`openstack.exceptions.NotSupported: The placement service for <vip>:RegionOne exists but does not have
any supported versions.`), so the restarts of step 4 fail with Placement stopped. If you stopped it, start it before
step 4.
**Any:** `systemctl stop <api services>` or the equivalent HAProxy change.

Leave Keystone running. The verification gate before reopening needs tokens, and with the other APIs stopped
Keystone alone gives users nothing they can act on. Stop it only if Keystone itself, or its database access, is
part of the failure; you will then need to start it first in step 4 before any verification is possible.

### Step 2: recover the message bus

> 🛑 **Destructive.** Purging or resetting RabbitMQ discards every in-flight RPC message: operations in progress
> (builds, deletes, migrations) are lost and their instances will need [stale-record clean-up](../lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md).
> Affects every service. Precondition: the APIs are stopped (step 1), so no new requests accumulate; you have
> established the cluster is partitioned or its state is unrecoverable by rejoining nodes. With Khepri, the
> Kolla-Ansible command can fail and leave one node stopped and outside the cluster (on the lab; recovery in 2
> below). Cannot be undone.

First, establish the state on every node (`rabbitmqctl cluster_status`, `check_alarms`, which store the cluster
uses, which queue types). With Khepri, a partition heals by itself once the network is back (on the lab, the three
nodes were running together within 15 s of the repair): check `cluster_status` on every node again before deciding on
a reset. If a clean rejoin of one side is possible, prefer it. Otherwise, the incident path:

1. Stop the RabbitMQ application on every node (`rabbitmqctl stop_app`), the last node stopped first on restart.
2. Reset or purge according to your deployment tool's procedure. **Kolla-Ansible:** `kolla-ansible rabbitmq-reset-state`
   resets the cluster and recreates it. **Any:** `rabbitmqctl force_reset` on the nodes, then rejoin, or purge the
   queues one by one (`rabbitmqctl purge_queue <queue>`) if the cluster itself is healthy.
   With Khepri (the default of a new RabbitMQ 4.2 cluster, as Kolla-Ansible 2026.1 deploys it), `force_reset` is
   refused (`Forced reset is unsupported with Khepri`). On the lab, `kolla-ansible rabbitmq-reset-state` failed: the
   fallback `rabbitmqctl reset` timed out on one node, which left the cluster with its application stopped, and the
   two others waited for 3 nodes and failed (`Not enough nodes joined ...'s cluster`); users, vhosts and queues were
   kept on the two others. That node came back with `rabbitmqctl reset`, `rabbitmqctl join_cluster
   rabbit@<other-node>`, `rabbitmqctl start_app`.
3. Start the application on every node and confirm `cluster_status` shows one cluster, no partitions, no alarms.
4. After a node was reset or re-joined, check that every quorum queue and stream has a member on it
   (`rabbitmqctl list_queues name type members`). On the lab, the re-joined node was a member of none: quorum queues
   ran on two members, and every client connected to that node failed on the streams with `PRECONDITION_FAILED -
   stream queue '<queue>' in vhost '/' does not have a running replica on the local node` (the Neutron agents of one
   compute: a new instance got no DHCP). Add the members back.

   > ⚠️ **State-changing.** Adds new members to the streams and quorum queues: each copies the queue's data, a
   > replication load on the cluster. Precondition: the node is running and listed as joined in `cluster_status`.
   > Undo: `rabbitmq-queues shrink rabbit@<node>` and `rabbitmq-streams delete_replica -p / <stream> rabbit@<node>`.

   `rabbitmq-streams add_replica -p / <stream> rabbit@<node>` for each stream, and
   `rabbitmq-queues grow rabbit@<node> all` for the quorum queues (Kolla-Ansible's guide also points to
   `contrib/ops/rabbitmq/rabbitmq-repair-stream-replicas.sh`, not shipped in the kolla-ansible package). An instance
   that booted meanwhile needs a hard reboot to get its DHCP lease.

The lab run of this step happened after the partition had healed, outside the step's precondition, to test the
commands; what it observed is in 2 and 4 above.

When RabbitMQ returns, every agent on every host reconnects at once; on a large platform that reconnection storm is
itself a load to expect, and consumers may take minutes to reappear on queues.

### Step 3: recover the database

> 🛑 **Destructive if done wrong.** Bootstrapping Galera from the wrong node discards the transactions the others
> had committed. Precondition: identify the most advanced node (`grastate.dat`: `seqno` and `safe_to_bootstrap`;
> after an unclean stop, run `mysqld --wsrep-recover` on each node to find the highest seqno). Backup first if the
> data directories allow it.

If the cluster is merely degraded (one node non-Primary), restart that node and let it SST/IST from the others. If
the whole cluster is non-Primary, follow the Galera crash-recovery procedure or your tool's:
**Kolla-Ansible:** `kolla-ansible mariadb-recovery`.
Confirm on every node: `wsrep_cluster_size` is full, `wsrep_cluster_status` is `Primary`,
`wsrep_local_state_comment` is `Synced`.

### Step 4: restart the OpenStack services that hold bus connections

> ⚠️ **State-changing.** Restarting conductors, schedulers, `nova-compute` and Neutron agents re-establishes
> clean RPC connections. Agent restarts can briefly interrupt data-plane changes on the host; `nova-compute`
> restarts do not affect running instances.

Restart conductors and schedulers on the controllers, then `nova-compute` and, with an agent-based backend
(ML2/OVS), the Neutron agents, in batches. With OVN there are no Neutron agents on the bus: `ovn-controller` talks to
the OVN databases, not to RabbitMQ, and does not need a restart for this.

## Verification, before reopening

All three must hold. Checks 2 and 3 need `nova-api`, `neutron-server` and `glance-api` running (and `cinder-api` for
boot from volume), reachable through the internal VIP: Nova reaches Glance and Neutron through it.

> ⚠️ **State-changing.** On Kolla-Ansible, starting those units (`sudo systemctl start
> kolla-<container>-container.service`, for the units stopped in step 1) puts them back behind HAProxy, which opens
> them to users before the gate passes: with a single VIP there is no internal-only path. That is what the lab did:
> the APIs were started, then checks 2 and 3 run. Precondition: check 1 holds. Undo: stop the units again.

```bash
# 1. Shared infrastructure whole
rabbitmqctl cluster_status                                   # one cluster, no partitions, no alarms
rabbitmqctl list_queues name messages consumers | awk '$3==0 && $2>0'   # empty: every queue has consumers
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size'; SHOW STATUS LIKE 'wsrep_cluster_status';"
#    Kolla-Ansible: sudo docker exec mariadb mariadb -uroot -p<database_password> -e "..." (no mysql in the image)

# 2. Every compute service and network agent reporting again. These need nova-api and neutron-server (see below);
#    read the output: with the API stopped, the command prints a 503 and grep still prints 0.
openstack compute service list --service nova-compute | grep -c down     # 0
openstack network agent list -f value -c Alive | grep -vc True          # 0 = every agent alive (agent-based backends)

# 3. End-to-end: create, reach, delete a test instance
openstack server create --flavor <flavor> --image <image> --network <network> --wait reopen-test
openstack server add floating ip reopen-test <fip>   # or test from a host on the network
ping -c 3 <address>
openstack server delete --wait reopen-test
```

Only then re-enable the API backends (**Kolla-Ansible:** `systemctl start` the units stopped in step 1, or take the
backends out of `MAINT`) and announce the reopening. Expect a burst of retried requests from users and
automation.

## Afterwards

- Establish the cause. In the source incident it was a network partition within the RabbitMQ cluster; a known
  failure mode with known mitigations, from the partition-handling setting on the versions that still have one to
  the network design that prevents the split. An incident review that stops at "we restarted RabbitMQ" guarantees
  a repeat.
- Clean up what the purge interrupted: [stale records](../lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md).
- Record the decision: who stopped, why, what was protected, what condition reopened. Next time the authority
  question is already answered.

## Validity and limits

Derived from one real incident on an HA control plane; release, RabbitMQ and Galera versions not recorded. The
RabbitMQ steps depend heavily on version (Mnesia versus Khepri, quorum versus classic queues): read the RabbitMQ
documentation for your version before resetting anything. Deployment-tool commands are given for Kolla-Ansible;
other tools have their own recovery commands. This is the heaviest option; use it when lighter ones (rejoin one
node, restart one service) do not apply.

Lab (Kolla-Ansible 22.2.0, 2026.1, 3 controllers, RabbitMQ 4.2.9 with Khepri, quorum queues and stream fanout, Galera
with ProxySQL, ML2/OVS, no Cinder, 2026-10-27): a partition of one RabbitMQ node made `server create` answer 504
and left the record in `BUILD`; steps 1, 2 (`kolla-ansible rabbitmq-reset-state`, run after the partition had
healed, to test it), 3 (read-only: Galera was healthy) and 4, the verification and the reopening were run in this
order, with the results written in the steps above; the test instance was reachable after the stream replicas were
repaired and a hard reboot. Not run: `rabbitmqctl stop_app` by hand, `force_reset` or `purge_queue` outside
Kolla-Ansible, a Galera recovery, HAProxy `MAINT`, OVN, Mnesia.

## Related

- [VM creation fails platform-wide: check the shared dependencies first](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md).
- [Stale records](../lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md).
- Source: *OpenStack, the Day After Tomorrow* (S. Zaouam), chapters 1, 10, 11 and 19.

---

*Provenance: written by agent:ozarland-author, from the book *OpenStack, the Day After Tomorrow*.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*
