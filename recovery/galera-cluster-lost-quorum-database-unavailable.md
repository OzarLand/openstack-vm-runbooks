---
title: Every API fails on database errors. The Galera cluster lost quorum, or stopped on every node
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [mariadb, galera, kolla-ansible, nova, neutron, keystone]
deployment_methods: [kolla-ansible, any]
environment:
  topology: HA control plane, Galera cluster of three or more MariaDB nodes
  network: any
  storage: any
  os: any
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible multinode, 3 controllers with ProxySQL, 2 computes", date: 2026-10-26, note: "full procedure on a disposable lab: case A (one node stopped), case B (port 4567 dropped on every node, then quorum reset with wsrep_provider_pc_bootstrap), case C (power loss of every node, mariadb-recovery 4.5 min, API containers restarted); mariadb_recover_inventory_name and Any variants not run" }
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/user/operating-kolla.html
  - https://docs.openstack.org/kolla-ansible/2026.1/admin/mariadb-backup-and-restore.html
  - https://galeracluster.com/library/documentation/crash-recovery.html
  - https://galeracluster.com/library/documentation/quorum-reset.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Every API fails on database errors. The Galera cluster lost quorum, or stopped on every node

## Symptom

Every OpenStack API, or most of them, answers with HTTP 500 or 503, or hangs. The service logs show database
errors, for example:

```text
(pymysql.err.OperationalError) (1047, 'WSREP has not yet prepared node for application use')
(pymysql.err.OperationalError) (2013, 'Lost connection to MySQL server during query')
(pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on ...")
```

With ProxySQL in front of Galera (Kolla-Ansible 2026.1 deploys it by default with MariaDB), the services see
ProxySQL's errors instead (observed on a lab):

```text
(pymysql.err.OperationalError) (9001, 'Max connect timeout reached while reaching hostgroup 0 after 10000ms')
(pymysql.err.OperationalError) (2013, 'Lost connection to MySQL server during query')
(pymysql.err.OperationalError) (1180, 'Error while appending streaming replication fragment(provider status: Not connected to Primary Component)')
```

`1047 WSREP has not yet prepared node for application use` is what a client connected directly to a node gets.

On the MariaDB nodes, one of two pictures:

- the `mariadb` containers run, but `wsrep_cluster_status` is `non-Primary` on every node (the cluster lost quorum);
- the `mariadb` containers are stopped or restarting on every node (after a power loss or a reboot of every
  controller), and a `kolla-ansible deploy` stops with `MariaDB cluster exists but is stopped. Please start it using
  kolla-ansible mariadb-recovery`.

## Impact

The whole control plane. Nova, Neutron, Cinder, Glance, Keystone and Placement keep their state in this database:
no instance, network or volume can be created, changed or deleted, and authentication fails if Keystone cannot read
its tables. Running instances keep running and keep their network (see
[control plane down](../diagnostics/control-plane-down-what-still-works-for-instances.md)).

The data is at risk only during the recovery: bootstrapping the cluster from a node that is not the most advanced
one discards the transactions only the other nodes had.

## Quick checks

Read-only. On **every** MariaDB node:

**Kolla-Ansible:**

```bash
# 1. Is the container up?
sudo docker ps -a --filter name=mariadb --format '{{.Names}} {{.Status}}'

# 2. If it is up: the node's view of the cluster (database_password from passwords.yml)
sudo docker exec -it mariadb mariadb -uroot -p -e "SHOW STATUS WHERE Variable_name IN
  ('wsrep_cluster_status','wsrep_cluster_size','wsrep_local_state_comment','wsrep_ready','wsrep_last_committed')"

# 3. The last lines of its log
sudo tail -n 50 /var/log/kolla/mariadb/mariadb.log
```

**Any:** the same `SHOW STATUS` query with the `mariadb` (or `mysql`) client on each node, and the MariaDB error log.

| What the nodes show | Case |
|---|---|
| Every node `Primary`, `Synced`, same `wsrep_cluster_size` | The cluster is healthy: look elsewhere ([shared dependencies](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md)) |
| One node `non-Primary` or stopped, the others `Primary` | Case A: one node left a cluster that kept quorum |
| Every running node `non-Primary`, `wsrep_ready` `OFF` | Case B: the cluster lost quorum, the processes still run |
| No node running, or containers restarting in a loop | Case C: the cluster stopped on every node |

## Diagnosis

### How Galera decides who may write

Galera accepts writes only in a **Primary component**: a group of nodes that holds a majority of the last known
membership. When nodes disappear without leaving cleanly (network split, crash, power loss) and the remaining nodes
are not a majority, they switch to `non-Primary` and refuse queries, to avoid two sides writing different data
(Galera crash recovery and quorum reset guides). With three nodes, one node lost leaves two, a majority; two lost at
once leaves one, not a majority, and that node stops serving.

Each node records the last transaction it applied (its **seqno**). To start a new Primary component without losing
data, start it from the node with the highest seqno, then let the others join and copy what they miss (IST or SST).

### What Kolla-Ansible does (2026.1)

`kolla-ansible mariadb-recovery -i <inventory>` (`ansible/roles/mariadb/tasks/recover_cluster.yml`) is meant for "a
completely stopped mariadb cluster" (operating guide). Without options it:

1. stops the `mariadb` container **on every MariaDB host**;
2. runs MariaDB with `--wsrep-recover` on each host, and reads the `Recovered position <uuid>:<seqno>` line from
   `/var/log/kolla/mariadb/mariadb.log`;
3. picks the host with the highest seqno, sets `safe_to_bootstrap: 1` in its `grastate.dat`, and starts it with
   `--wsrep-new-cluster` (`pc.bootstrap=yes` in its `galera.cnf`);
4. starts the other hosts, waits until each is `Synced`, then restarts the first one with its normal configuration.

With `-e mariadb_recover_inventory_name=<host>` it skips steps 1 and 2 and bootstraps from the host you name. The
backup and restore guide uses that form after a restore, with the first node of the cluster.

## Possible causes

### Cause 1: one node failed (case A)

A crash, a reboot or a network problem on one controller. The cluster kept a majority and serves; the APIs fail only
if they were routed to that node, which ProxySQL or the load balancer stops doing once that node fails its checks.

### Cause 2: a network split or several nodes lost at once (case B)

No side holds a majority: a network problem between the controllers, two of three controllers lost together, or a
two-node cluster that lost one node.

### Cause 3: every node stopped (case C)

A power loss, every controller rebooted, or `kolla-ansible stop`. Galera does not restart a stopped cluster on its
own: a node that was not stopped last has `safe_to_bootstrap: 0`, and none knows whether another node has newer data.

## Resolution

Before any state-changing step, take a backup. In case A, the cluster still has a Primary component, and
`kolla-ansible mariadb-backup -i <inventory>` works (see the backup and restore guide) if `enable_mariabackup: "yes"`
is set in `globals.yml`. Without it, the command ends with `failed=0` and takes no backup (every task is skipped):
check that a backup file was written. In cases B and C it does not:
`mariadb-backup` needs a node in a Primary component, so copy each node's `mariadb` Docker volume instead. Stop the
container first with `sudo systemctl stop kolla-mariadb-container.service` on each node, not `docker stop` (systemd
starts it again after `RestartSec`, 11 s with the Kolla defaults, and `mysqld` would write under the copy), check
with `sudo docker ps -a` that it stays stopped, and only then copy the volume. Then stop new requests at the APIs if
your procedure allows it ([reopening provisioning](reopening-provisioning-after-control-plane-incident.md)).

### For cause 1: let the failed node rejoin

Fix what stopped the node (disk, network, host), then start its container. It joins the Primary component and
copies what it missed.

> ⚠️ **State-changing.** Starts MariaDB on that node; it receives the missing transactions from a donor node, which
> slows the donor while the copy runs. Undo: `sudo systemctl stop kolla-mariadb-container.service`.

On Kolla-Ansible 2026.1, each container has a systemd unit (`kolla-<container>-container.service`, `Restart=always`,
`RestartSec` 11 s with the Kolla defaults, 59 s on the lab, which sets `docker_client_timeout: 600`): a container
stopped with `docker stop` is started again by systemd after that delay. To keep a
node's MariaDB stopped while you repair the host, use `sudo systemctl stop kolla-mariadb-container.service`, and
`sudo systemctl start kolla-mariadb-container.service` to start it.

```bash
sudo systemctl start kolla-mariadb-container.service
sudo docker exec -it mariadb mariadb -uroot -p -e "SHOW STATUS LIKE 'wsrep_local_state_comment'"   # Joining, then Synced
```

Do not run `mariadb-recovery` for this case: it stops the healthy nodes too.

### For cause 2: restore connectivity first

If the split comes from the network, repair the network: once the nodes see each other again, they form a Primary
component again on their own, and nothing else is needed. Check with quick check 2.

If the missing nodes cannot come back soon, the Galera quorum reset guide makes the most advanced running node
Primary again: compare `wsrep_last_committed` on the running nodes, then on the node with the highest value only:

> 🛑 **Destructive.** Declares this node's data the reference. Transactions that only a node outside this component
> had are lost when that node rejoins and copies from this one. Run it on one node only: on two sides of a split,
> it creates two clusters that write different data. Precondition: the other side is down or cut off for sure, and
> this node has the highest `wsrep_last_committed` of all reachable nodes. Cannot be undone.

**Kolla-Ansible:**

```bash
sudo docker exec -it mariadb mariadb -uroot -p -e "SET GLOBAL wsrep_provider_pc_bootstrap=ON"
# once the node is Primary, reset the variable (the bootstrap is not repeated)
sudo docker exec -it mariadb mariadb -uroot -p -e "SET GLOBAL wsrep_provider_pc_bootstrap=OFF"
```

Observed on the Kolla-Ansible 2026.1 image (MariaDB 11.4.13): the Galera provider options are plugin variables
(`wsrep_provider_pc_*`): `SET GLOBAL wsrep_provider_options='pc.bootstrap=YES'` fails with
`ERROR 1210 (HY000): wsrep_provider_options cannot be changed while the wsrep-provider plugin is loaded`.

**Any:** `SET GLOBAL wsrep_provider_pc_bootstrap=ON;` where `SHOW GLOBAL VARIABLES LIKE 'wsrep_provider_pc%'` lists it,
otherwise `SET GLOBAL wsrep_provider_options='pc.bootstrap=YES';`.

The other running nodes then join this component. Nodes that were cut off join when they come back.

### For cause 3: Kolla-Ansible's recovery

Precondition: every controller is up, with its network and the `mariadb` Docker volume intact. Leave the volumes
alone: the recovery reads the seqno from them.

> 🛑 **Destructive.** Stops MariaDB on every MariaDB host, then rebuilds the cluster from the host with the highest
> seqno. If a host's seqno cannot be read (its log has no `Recovered position` line), the run may fail and leave
> MariaDB stopped everywhere: read every host's `/var/log/kolla/mariadb/mariadb.log` before running it again. All
> database clients are cut off while it runs. Cannot be undone;
> keep a backup of each host's `mariadb` volume if you can: stop the container with
> `sudo systemctl stop kolla-mariadb-container.service` (not `docker stop`, which systemd undoes), check that it stays
> stopped, then copy the volume directory.

```bash
kolla-ansible mariadb-recovery -i <inventory>
```

To choose the bootstrap host yourself (the backup and restore guide uses this form after a restore):

> 🛑 **Destructive.** With `mariadb_recover_inventory_name`, the playbook skips the stop, the `--wsrep-recover` run
> and the seqno comparison, and bootstraps the cluster from the host you name. If that host is not the most advanced,
> the transactions only the other hosts had are lost when they join it and copy from it. Precondition: every `mariadb`
> container is stopped, and you know this host is the most advanced: its `seqno:` in
> `<volumes path>/mariadb/_data/grastate.dat` (`/var/lib/docker/volumes/mariadb/_data/grastate.dat` by default) is the
> highest of all hosts. After a crash or a power loss, `grastate.dat` usually says `seqno: -1`: then you cannot tell,
> and the automatic form above is the one to use. Cannot be undone.

```bash
kolla-ansible mariadb-recovery -i <inventory> -e mariadb_recover_inventory_name=<host>
```

After a power loss of every controller, the API containers that started while the database was unavailable may
not recover on their own once the cluster is back (observed on a lab: `nova_api`, `nova_metadata`, `placement_api`
and `neutron_server` stayed `unhealthy`, their uWSGI log said `unable to load app 0` then `no python application
found`, HAProxy answered `503 No server is available to handle this request`, and `nova_conductor` and
`nova_scheduler` crash-looped with `The placement service for <vip>:RegionOne exists but does not have any supported
versions.`, while the unhealthy uWSGI services answered HTTP 500 to every request). Check every controller with
`sudo docker ps --filter health=unhealthy` and restart what it lists:

> ⚠️ **State-changing.** Restarts the API containers; requests in flight on that controller fail. Undo: none needed.

```bash
sudo systemctl restart kolla-<container>-container.service   # for each of them, e.g. placement_api, nova_api,
                                                              # nova_metadata, neutron_server
```

**Any:** the Galera crash recovery guide: find the node with the highest seqno (`grastate.dat`, or
`mysqld --wsrep-recover` when it says `-1`), set `safe_to_bootstrap: 1` on that node only, start it with
`--wsrep-new-cluster` (or `galera_new_cluster`), then start the others normally.

## Verification

```bash
# on every node: Primary, Synced, ON, and the same size (the number of nodes)
sudo docker exec -it mariadb mariadb -uroot -p -e "SHOW STATUS WHERE Variable_name IN
  ('wsrep_cluster_status','wsrep_local_state_comment','wsrep_ready','wsrep_cluster_size')"
sudo docker ps --filter name=mariadb --format '{{.Names}} {{.Status}}'        # healthy
sudo docker ps --filter health=unhealthy --format '{{.Names}} {{.Status}}'     # empty on every controller

# from a host with admin credentials
openstack token issue
openstack server list --all-projects --limit 5
openstack compute service list
```

Then check the services that may have failed while the database was gone, before reopening provisioning:
[reopening provisioning after a control-plane incident](reopening-provisioning-after-control-plane-incident.md).

## Lab observations

Kolla-Ansible 22.2.0 (2026.1), three controllers with Galera and ProxySQL, two computes, MariaDB 11.4.13, Galera
26.4.27, 2026-10-26:

- **Case C** (every VM of the lab powered off, then on): each `mariadb` container ran `--wsrep-recover`, logged
  `No nodes coming from primary view, primary view is not possible`, then `failed to reach primary view` and
  `Aborting` about 30 s later, and was restarted by systemd a minute later, in a loop. `grastate.dat` showed
  `seqno: -1`, `safe_to_bootstrap: 0` on all three. `kolla-ansible deploy --tags mariadb` failed with the message
  quoted in *Symptom*. `kolla-ansible mariadb-recovery` took 4 min 34 s, found the same seqno on every node,
  bootstrapped from one, and the others joined by SST. The API containers then had to be restarted (see cause 3).
- **Case A** (`docker stop mariadb` on the ProxySQL writer): the other two stayed `Primary`, size 2; ProxySQL moved
  the writer after 14 s; no API error was seen. `docker start` brought the node to `Synced` in 18 s (IST).
- **Case B** (Galera port 4567 dropped on every node): every node `non-Primary`, `wsrep_ready` `OFF` after 9 s; the
  APIs returned 502. Once the port was open again, the Primary component formed again on its own within 3 s. The
  quorum reset with `wsrep_provider_pc_bootstrap=ON` on one node made it Primary at once; the others joined by IST
  when the port was opened.

## Validity and limits

- Written from Kolla-Ansible 2026.1 (`recover_cluster.yml`, `lookup_cluster.yml`, `galera.cnf.j2`, the operating
  guide and the MariaDB backup and restore guide) and the Galera crash recovery and quorum reset guides. Cases A, B
  (with the quorum reset) and C run on a disposable Kolla-Ansible 2026.1 lab with ProxySQL (see *Lab observations*);
  the `mariadb_recover_inventory_name` form and the **Any** variants were not run.
- Assumes Docker; with Podman, replace `docker` with `podman`.
- Not covered: a single-node MariaDB (Kolla-Ansible restarts it without recovery), MariaDB shards
  (`mariadb_shard_group`), restoring from a backup (see the backup and restore guide), ProxySQL problems with a
  healthy cluster, and Galera arbitrators.

## Related

- [VM creation fails platform-wide. Check the shared dependencies first](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md):
  how to tell the database from the message bus and the other shared parts.
- [Reopening provisioning after a control-plane incident](reopening-provisioning-after-control-plane-incident.md)
- [Control plane down. What still works for running instances](../diagnostics/control-plane-down-what-still-works-for-instances.md)
- Kolla-Ansible 2026.1: [operating Kolla](https://docs.openstack.org/kolla-ansible/2026.1/user/operating-kolla.html),
  [MariaDB backup and restore](https://docs.openstack.org/kolla-ansible/2026.1/admin/mariadb-backup-and-restore.html)
- Galera: [crash recovery](https://galeracluster.com/library/documentation/crash-recovery.html),
  [quorum reset](https://galeracluster.com/library/documentation/quorum-reset.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation and source and the Galera
documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=recovery/galera-cluster-lost-quorum-database-unavailable.md&title=Feedback%3A+recovery/galera-cluster-lost-quorum-database-unavailable.md):
your release, your deployment method, and what happened.
