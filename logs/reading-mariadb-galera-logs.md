---
title: Reading the MariaDB and Galera logs, and the oslo.db lines that go with them
kind: diagnostic
status: lab-verified
openstack_releases: ["2026.1"]
components: [mariadb, proxysql, haproxy, nova, neutron, keystone]
deployment_methods: [kolla-ansible, any]
environment:
  topology: multinode, 3 database nodes
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, 3 controllers + 2 computes, Galera (MariaDB 11.4.13, Galera 26.4.27) behind ProxySQL", date: 2026-10-27, note: "read-only commands run (status query without -it); one node cut from the others on port 4567 for 1 min 45 s and rejoined by IST; clean stop/restart of one node from the #80 run. Not seen: crash, whole-cluster quorum loss, bootstrap errors, full SST or SST failure, donor state, errors 1040/1213, any oslo.db line" }
upstream_refs:
  - https://github.com/codership/galera/tree/release_26.4.27
  - https://github.com/MariaDB/server/tree/mariadb-11.4.13/sql
  - https://opendev.org/openstack/oslo.db/src/branch/stable/2026.1/oslo_db/sqlalchemy/exc_filters.py
  - https://opendev.org/openstack/oslo.db/src/branch/stable/2026.1/oslo_db/sqlalchemy/engines.py
  - https://opendev.org/openstack/kolla-ansible/src/branch/stable/2026.1/ansible/roles/mariadb/templates/galera.cnf.j2
  - https://docs.openstack.org/kolla-ansible/latest/reference/databases/mariadb-guide.html
  - https://mariadb.com/kb/en/galera-cluster-status-variables/
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Reading the MariaDB and Galera logs, and the oslo.db lines that go with them

Every OpenStack service depends on the database cluster. When Galera loses a node, loses quorum or resynchronises
a node, the services log `DBConnectionError` or deadlocks, and the explanation is in the MariaDB error log of each
database node, in the lines that start with `WSREP:`. This page reads both sides. Galera messages are quoted from
the Galera 26.4.27 source, MariaDB messages from MariaDB 11.4.13 (the series Kolla 2026.1 installs, from the
`mariadb-server/11.4` repositories in `kolla/template/repos.yaml`; the patch level in your image may differ), client
messages from `oslo.db` 2026.1.

Recovering a cluster that lost quorum is a runbook, not this page.

## Where the log is, and its format

**Kolla-Ansible:** `log_error=/var/log/kolla/mariadb/mariadb.log` (`galera.cnf.j2`), on each database node (the
controllers by default). If you installed the `wsrep-notify.sh` example of the
[Kolla-Ansible MariaDB guide](https://docs.openstack.org/kolla-ansible/latest/reference/databases/mariadb-guide.html),
cluster events also go to `/var/log/kolla/mariadb/galera_cluster_events.log`.

**Any:** the `log_error` setting of the server, or the journal of the `mariadb` unit.

MariaDB writes each line as (`print_buffer_to_file` in `sql/log.cc`):

```text
<YYYY-MM-DD> <HH:MM:SS> <thread id> [<Note|Warning|ERROR>] <message>
```

Messages from the Galera library are passed to this log with the prefix `WSREP: ` (`wsrep_mysqld.cc`). Times are the
host's local time, at one-second resolution. Some Galera messages span several lines (`Quorum results:`, the view
of the cluster); read the indented lines after the first.

```bash
# Kolla-Ansible, on a database node
grep -n 'WSREP' /var/log/kolla/mariadb/mariadb.log | tail -50
grep -nE '\[(ERROR|Warning)\]' /var/log/kolla/mariadb/mariadb.log | tail -30
```

The state now, rather than the history, is in the status variables (read-only). On Kolla-Ansible, run the query on
each database node with `docker exec -it mariadb mariadb -u root -p` (the root password is `database_password` in
`passwords.yml`, on the deploy host; drop `-it` when the command is not run from a terminal):

```sql
SHOW GLOBAL STATUS WHERE Variable_name IN
  ('wsrep_cluster_status','wsrep_cluster_size','wsrep_local_state_comment','wsrep_ready','wsrep_connected');
```

`wsrep_cluster_status` `Primary` and `wsrep_local_state_comment` `Synced` on every node is the healthy state.

## Node state: the Shifting lines

Galera logs each change of the node's state (`gcs/src/gcs.cpp`):

```text
WSREP: Shifting <old> -> <new> (TO: <seqno>)
```

The states are `SYNCED`, `JOINED`, `DONOR/DESYNCED`, `JOINER`, `PRIMARY`, `OPEN`, `CLOSED`, `DESTROYED` and `ERROR`.
`TO` is the global transaction sequence number at that moment.

| Transition | What it means for OpenStack |
|---|---|
| `OPEN -> PRIMARY` | The node joined a primary component (a quorum). |
| `PRIMARY -> JOINER` | It needs a state transfer (IST or SST) to catch up. It does not serve queries until it is synced. |
| `JOINER -> JOINED`, then `JOINED -> SYNCED` | Caught up; serving again. |
| `SYNCED -> DONOR/DESYNCED` | This node is sending a state transfer to a joiner (Kolla-Ansible's default `mariadb_wsrep_sst_method` is `mariabackup`). It may serve clients more slowly; it refuses them only if `wsrep_sst_donor_rejects_queries` is on. |
| `DONOR/DESYNCED -> JOINED -> SYNCED` | The donation ended. |
| `... -> OPEN` or `... -> CLOSED` | The node left the primary component (quorum lost, or shut down). |

## Membership and quorum

| Line (message text) | Level | What it means |
|---|---|---|
| `WSREP: Quorum results:` followed by `version = …`, `component = PRIMARY` or `NON-PRIMARY`, `conf_id = …`, `members = <joined>/<total> (joined/total)`, `act_id = …`, … | Note | A new view of the cluster was computed. `component = NON-PRIMARY` means **this node has no quorum**: it refuses queries. `members` says how many nodes it sees. On the lab, a node cut from the others logged no `Quorum results` block, only `Received NON-PRIMARY.` (see below). |
| `WSREP: <self> connection to peer <uuid> with addr <address> timed out, no messages seen in <period>, socket stats: …` | Note | The group communication link to one peer was silent for `gmcast.peer_timeout`. The network between database nodes failed, or the peer is stuck. |
| `WSREP: <self> suspecting node: <uuid>` / `suspected node without join message, declaring inactive` (or `detected inactive node: <uuid>`) | Note | The node suspects, then declares, a peer gone. A new view follows. On a lab network cut, the second form appeared, and `detected inactive node` did not. |
| `WSREP: forgetting <uuid> (<address>)` | Note | The peer is removed from the list of known members. |
| `WSREP: Member <idx>.<segment> (<name>) synced with group.` | Note | That member reached `SYNCED`. |
| `WSREP: failed to open gcomm backend connection: 110: failed to reach primary view…` | ERROR | At start, the node could not join a primary component within the timeout: no other node with quorum was reachable. |
| `WSREP: It may not be safe to bootstrap the cluster from this node. It was not the last one to leave the cluster and may not contain all the updates. To force cluster bootstrap with this node, edit the grastate.dat file manually and set safe_to_bootstrap to 1 .` | ERROR | Someone (or an automation) tried to bootstrap a new cluster from a node that was not the last to stop. Galera refuses, to avoid losing transactions. Do not force it before comparing the nodes' positions. |

Sources: `gcs/src/gcs_group.cpp`, `gcomm/src/gmcast.cpp`, `gcomm/src/evs_proto.cpp`, `gcs/src/gcs_gcomm.cpp`,
`galera/src/replicator_smm.cpp` (Galera 26.4.27).

**Captured on a lab** (Kolla-Ansible 2026.1, MariaDB 11.4.13, Galera 26.4.27, 3 nodes). One node cut from the two
others on port 4567 for 1 min 45 s. The cut node (controller-3) logged no `Quorum results:` block while it had no
quorum: it logged `view(view_id(NON_PRIM, …))`, `New COMPONENT: primary = no`, `Received NON-PRIMARY.` and `Non-primary
view`, then `timed out` lines per peer until the network came back:

```text
2026-10-27 17:44:07 0 [Note] WSREP: (aaaaaaaa-0003, 'tcp://192.0.2.13:4567') connection to peer aaaaaaaa-0001 with addr tcp://192.0.2.11:4567 timed out, no messages seen in PT3S, socket stats: rtt: 13812 rttvar: …
2026-10-27 17:44:09 0 [Note] WSREP: (aaaaaaaa-0003, 'tcp://192.0.2.13:4567') reconnecting to aaaaaaaa-0001 (tcp://192.0.2.11:4567), attempt 0
2026-10-27 17:44:09 0 [Note] WSREP: evs::proto(aaaaaaaa-0003, OPERATIONAL, view_id(REG,aaaaaaaa-0001,6)) suspecting node: aaaaaaaa-0001
2026-10-27 17:44:09 0 [Note] WSREP: evs::proto(aaaaaaaa-0003, OPERATIONAL, view_id(REG,aaaaaaaa-0001,6)) suspected node without join message, declaring inactive
2026-10-27 17:44:10 0 [Note] WSREP: New COMPONENT: primary = no, bootstrap = no, my_idx = 0, memb_num = 1
2026-10-27 17:44:10 0 [Note] WSREP: Received NON-PRIMARY.
2026-10-27 17:44:10 0 [Note] WSREP: Shifting SYNCED -> OPEN (TO: 4315)
2026-10-27 17:44:10 2 [Note] WSREP: Non-primary view
2026-10-27 17:44:10 2 [Note] WSREP: Server status change synced -> connected
```

The two other nodes kept quorum (`component = PRIMARY`, `members = 2/2 (joined/total)` at 17:44:11) and logged the
cut node under `partitioned`, then `forgetting` (in the run's capture of controller-1). A clean stop of a node (`systemctl stop
kolla-mariadb-container.service`, same lab) logs no `timed out`, `suspecting` or `inactive` line: the peers log
`forgetting <uuid>` and a view with the node under `partitioned` in the same second. The status variables on the cut
node read `wsrep_cluster_status` `non-Primary`, `wsrep_local_state_comment` `Initialized`, `wsrep_ready` `OFF`.

Which recovery applies to which case (one node lost, quorum lost, every node stopped) and how to compare positions
safely: [Galera cluster lost quorum or stopped on every node](../recovery/galera-cluster-lost-quorum-database-unavailable.md)
is the main reference; reopening the platform afterwards is in
[reopening provisioning after a control-plane incident](../recovery/reopening-provisioning-after-control-plane-incident.md).

## State transfers: IST and SST

A node that rejoins needs the transactions it missed. If the donor still has them in its write-set cache it sends
only those (**IST**, incremental); otherwise it sends a full copy of the data (**SST**), which takes as long as
copying the whole database.

| Line (message text) | Level | Log | What it means |
|---|---|---|---|
| `WSREP: Member <idx>.<seg> (<joiner>) requested state transfer from '<donor or *any*>'. Selected <idx>.<seg> (<donor>)(<state>) as donor.` | Note | every node | A join started; the donor is named. |
| `WSREP: Requesting state transfer: success, donor: <idx>` | Note | joiner | The request was accepted. |
| `WSREP: Prepared IST receiver for <first>-<last>, listening at: <address>, waiting for connection to: <address>` | Note | joiner | An incremental transfer is possible. |
| `WSREP: IST received: <uuid>:<seqno>` | Note | joiner | The incremental transfer ended. |
| `WSREP: Running: '<sst script command>'` | Note | donor and joiner | A full SST started, with the configured method's script. |
| `WSREP: Rejecting client queries for the duration of SST.` | Note | donor | Logged only when `wsrep_sst_donor_rejects_queries` is on (it is off by default): the donor refuses clients during the SST. |
| `WSREP: SST succeeded for position <uuid>:<seqno>` | Note | joiner | The full transfer ended. |
| `WSREP: SST failed for position <uuid>:<seqno> initialized <n> server_state <state>` | ERROR | joiner | The transfer failed; the node will not join. The SST script's own messages just before this line give the reason (disk space, network, credentials of the SST user). |
| `WSREP: Requesting state transfer failed: <error>` | ERROR or Note | joiner | No donor could be selected (for example, all others are themselves joining). |

Sources: `gcs/src/gcs_group.cpp`, `galera/src/replicator_str.cpp`, `galera/src/ist.cpp` (Galera);
`sql/wsrep_sst.cc` (MariaDB).

Captured on a lab, the cut node rejoining 3 s after the network came back, by IST:

```text
2026-10-27 17:45:53 0 [Note] WSREP: Shifting OPEN -> PRIMARY (TO: 4484)
2026-10-27 17:45:53 2 [Note] WSREP: Prepared IST receiver for 4316-4484, listening at: tcp://192.0.2.13:4568, waiting for connection to: tcp://192.0.2.13:4568
2026-10-27 17:45:53 0 [Note] WSREP: Member 1.0 (controller-3) requested state transfer from '*any*'. Selected 2.0 (controller-2)(SYNCED) as donor.
2026-10-27 17:45:53 0 [Note] WSREP: Shifting PRIMARY -> JOINER (TO: 4484)
2026-10-27 17:45:53 2 [Note] WSREP: Requesting state transfer: success, donor: 2
2026-10-27 17:45:53 2 [Note] WSREP: Receiving IST: 169 writesets, seqnos 4316-4484
2026-10-27 17:45:53 2 [Note] WSREP: IST received: 00000000-0000-4000-8000-000000000001:4484
2026-10-27 17:45:53 0 [Note] WSREP: Shifting JOINER -> JOINED (TO: 4486)
2026-10-27 17:45:53 0 [Note] WSREP: Member 1.0 (controller-3) synced with group.
```

After a restart of the node (same lab), the joiner also logged `Running: 'wsrep_sst_mariabackup --role 'joiner' …'`
and `SST succeeded for position <uuid>:<seqno>` with its own saved position, then `Receiving IST: 204 writesets`: the
data came by IST. Read `Receiving IST` before concluding a full SST ran.

A node that keeps cycling between `JOINER` and `OPEN` with a new SST each time usually has an SST that fails, not a
quorum problem: read the lines before `SST failed`.

## Connections and client-visible errors

| Line or error | Where | What it means |
|---|---|---|
| `Aborted connection <id> to db: '<db>' user: '<user>' host: '<host>' (<reason>)` | MariaDB log, `[Warning]` | A client connection ended without a clean close: the client process died, a timeout (`Got timeout reading communication packets`), or a proxy closed it. Some are normal; a burst at one moment is a client-side or network event. |
| `Too many connections` (error 1040) | the client | `max_connections` reached. Kolla-Ansible sets `max_connections=10000` (`galera.cnf.j2`). |
| `WSREP has not yet prepared node for application use` (error 1047) | the client | The node is not in a primary component, or not synced: it refuses queries. The proxy (ProxySQL, or HAProxy) should not have sent the client there. |
| `Deadlock found when trying to get lock; try restarting transaction` (error 1213) | the client | An InnoDB deadlock, **or** a Galera certification conflict: MariaDB 11.4 reports a conflict with the same error (`wsrep_override_error(thd, ER_LOCK_DEADLOCK)`), so the two look identical to the client. |

Sources: `sql/share/errmsg-utf8.txt`, `sql/sql_parse.cc`, `sql/wsrep_high_priority_service.cc`, `sql/wsrep_mysqld.cc`
(MariaDB 11.4.13).

Captured on a lab: a query sent directly to the cut node, and the `Aborted connection` kind seen at node joins
(not an error):

```text
ERROR 1047 (08S01) at line 1: WSREP has not yet prepared node for application use
2026-10-27 13:15:56 12 [Warning] Aborted connection 12 to db: 'unconnected' user: 'unauthenticated' host: 'controller-2' (This connection closed normally without authentication)
```

## The oslo.db side, in the OpenStack services

`oslo.db` turns the MySQL client errors into its own exceptions (`exc_filters.py`), which is what you see in the
services' tracebacks:

| In the service log | Comes from | What it means |
|---|---|---|
| `DBConnectionError` with `(2002 …)`, `(2003 …)`, `(2006 …)`, `(2013 …)` or `(1047 …)` in the message | MySQL errors 2002/2003 (cannot connect), 2006 (server has gone away), 2013 (lost connection during query), 1047 (node not ready) | The service could not reach a usable database node through the proxy (ProxySQL by default on Kolla-Ansible). |
| `DBDeadlock` with `1213` and `Deadlock found when trying to get lock` | InnoDB deadlock, or a Galera certification conflict (they look the same; `oslo.db` also matches an older `Deadlock: wsrep aborted` text that MariaDB 11.4 no longer sends) | Usually retried by the service (`oslo_db.api.wrap_db_retry`); a flood means many nodes take writes for the same rows at once. |
| `SQL connection failed. <n> attempts left.` | `engines.py`, WARNING | The service is starting and cannot connect; it retries. |
| `DB exceeded retry limit.` with a traceback | `api.py`, ERROR | A retried operation kept failing. |
| `DBAPIError exception wrapped.` / `DB exception wrapped.` with a traceback | `exc_filters.py`, WARNING | A database error that `oslo.db` did not classify; read the wrapped error. |

**Which proxy is in front of the database.** In Kolla-Ansible 2026.1 the default is **ProxySQL**
(`mariadb_loadbalancer: proxysql`, `enable_proxysql: "{{ enable_mariadb }}"`). It sends writes to one node at a time
(`mysql_galera_hostgroups` with `max_writers: 1` in `proxysql.yaml.j2`) and logs to
`/var/log/kolla/proxysql/proxysql.log` on each load balancer node. On a lab, each ProxySQL logged, 5 to 9 s after
the network cut of one database node, `[WARNING] Galera: setting host 192.0.2.13:3306 offline because:
primary_partition=NO` (`MySQL_HostGroups_Manager.cpp`); no `DBConnectionError` reached the services. With `enable_proxysql: false`, HAProxy's `mariadb` backend does the same job: one active server, the others `backup`. Either way, a burst
of `DBConnectionError` in many services at the same second is usually the active node going away and the proxy
failing over: read the proxy's log and the database node logs at that second.

## Reading a database incident in order

1. **Status variables on every node:** which nodes are `Primary`/`Synced`, which are not.
2. **The last `Quorum results:` on each node:** `component` and `members`, with their times.
3. **What made a node leave:** `connection to peer … timed out`, `suspecting node`, a restart (`Shifting … -> CLOSED`).
4. **How it came back:** IST or SST, and whether the SST succeeded.
5. **Then the clients:** the first and last `DBConnectionError` in the services bracket the outage as they saw it.

## Validity and limits

- Every MariaDB and Galera message on this page is quoted from the Galera 26.4.27 and MariaDB 11.4.13 sources; the
  `oslo.db` table also names an older deadlock text that `oslo.db` still matches but MariaDB 11.4 no longer sends.
  The blocks marked "captured on a lab" come from Kolla-Ansible 2026.1 (MariaDB 11.4.13, Galera 26.4.27, ProxySQL,
  3 database nodes): a clean stop and restart of one node, and a network cut of one node (port 4567) for 1 min 45 s.
  Not seen on the lab: `failed to open gcomm backend connection`, the `safe_to_bootstrap` error, a full SST, `SST
  failed`, a donor state, error 1040, 1213 and every `oslo.db` line (ProxySQL kept the services away from the cut
  node).
- `connection to peer … timed out` names `gmcast.peer_timeout`; the related `evs.*` timeouts that decide when a node
  is declared inactive are not detailed here.
- The SST script messages (for `mariabackup`) are not covered.
- ProxySQL's own log lines are not described here; only its log path and hostgroup layout, from Kolla-Ansible's
  templates. HAProxy's `mariadb` backend applies only when ProxySQL is disabled.

## Related

- [Galera cluster lost quorum or stopped on every node](../recovery/galera-cluster-lost-quorum-database-unavailable.md) (the recovery)
- [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md)
- [VM creation fails platform-wide: check the shared dependencies first](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md)
- [Reopening provisioning after a control-plane incident](../recovery/reopening-provisioning-after-control-plane-incident.md)
- MariaDB Galera status variables: https://mariadb.com/kb/en/galera-cluster-status-variables/
- [Logs index](README.md)

---

*Provenance: written by agent:ozarland-author, from Galera 26.4, MariaDB 11.4, oslo.db 2026.1 and Kolla-Ansible
2026.1 source code and documentation. Reviewed in its pull request (see the page history). Checked by a human
operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Freading-mariadb-galera-logs.md&title=Feedback%3A+logs%2Freading-mariadb-galera-logs.md):
your release, your deployment method, and what happened.
