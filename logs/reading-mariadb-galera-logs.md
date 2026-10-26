---
title: Reading the MariaDB and Galera logs, and the oslo.db lines that go with them
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [mariadb, nova, neutron, keystone]
deployment_methods: [kolla-ansible, any]
environment:
  topology: multinode, 3 database nodes
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by: []
upstream_refs:
  - https://github.com/codership/galera/tree/release_26.4.27
  - https://github.com/MariaDB/server/tree/mariadb-11.4.13/sql
  - https://opendev.org/openstack/oslo.db/src/branch/stable/2026.1/oslo_db/sqlalchemy/exc_filters.py
  - https://opendev.org/openstack/oslo.db/src/branch/stable/2026.1/oslo_db/sqlalchemy/engines.py
  - https://opendev.org/openstack/kolla-ansible/src/branch/stable/2026.1/ansible/roles/mariadb/templates/galera.cnf.j2
  - https://docs.openstack.org/kolla-ansible/latest/reference/databases/mariadb-guide.html
  - https://mariadb.com/kb/en/galera-cluster-status-variables/
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Reading the MariaDB and Galera logs, and the oslo.db lines that go with them

Every OpenStack service depends on the database cluster. When Galera loses a node, loses quorum or resynchronises
a node, the services log `DBConnectionError` or deadlocks, and the explanation is in the MariaDB error log of each
database node, in the lines that start with `WSREP:`. This page reads both sides. Galera messages are quoted from
the Galera 26.4.27 source, MariaDB messages from MariaDB 11.4.13 (the series Kolla 2026.1 installs, from the
`mariadb-server/11.4` repositories in `kolla/template/repos.yaml`), client messages from `oslo.db` 2026.1.

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

The state now, rather than the history, is in the status variables (read-only):

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
| `WSREP: Quorum results:` followed by `version = …`, `component = PRIMARY` or `NON-PRIMARY`, `conf_id = …`, `members = <joined>/<total> (joined/total)`, `act_id = …`, … | Note | A new view of the cluster was computed. `component = NON-PRIMARY` means **this node has no quorum**: it refuses queries. `members` says how many nodes it sees. |
| `WSREP: <self> connection to peer <uuid> with addr <address> timed out, no messages seen in <period>, socket stats: …` | Note | The group communication link to one peer was silent for `gmcast.peer_timeout`. The network between database nodes failed, or the peer is stuck. |
| `WSREP: <self> suspecting node: <uuid>` / `detected inactive node: <uuid>` | Note | The node suspects, then declares, a peer gone. A new view follows. |
| `WSREP: forgetting <uuid> (<address>)` | Note | The peer is removed from the list of known members. |
| `WSREP: Member <idx>.<segment> (<name>) synced with group.` | Note | That member reached `SYNCED`. |
| `WSREP: failed to open gcomm backend connection: 110: failed to reach primary view…` | ERROR | At start, the node could not join a primary component within the timeout: no other node with quorum was reachable. |
| `WSREP: It may not be safe to bootstrap the cluster from this node. It was not the last one to leave the cluster and may not contain all the updates. To force cluster bootstrap with this node, edit the grastate.dat file manually and set safe_to_bootstrap to 1 .` | ERROR | Someone (or an automation) tried to bootstrap a new cluster from a node that was not the last to stop. Galera refuses, to avoid losing transactions. Do not force it before comparing the nodes' positions. |

Sources: `gcs/src/gcs_group.cpp`, `gcomm/src/gmcast.cpp`, `gcomm/src/evs_proto.cpp`, `gcs/src/gcs_gcomm.cpp`,
`galera/src/replicator_smm.cpp` (Galera 26.4.27).

Which recovery applies to which case (one node lost, quorum lost, every node stopped) and how to compare positions
safely: see the recovery notes in
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

A node that keeps cycling between `JOINER` and `OPEN` with a new SST each time usually has an SST that fails, not a
quorum problem: read the lines before `SST failed`.

## Connections and client-visible errors

| Line or error | Where | What it means |
|---|---|---|
| `Aborted connection <id> to db: '<db>' user: '<user>' (<reason>)` | MariaDB log, `[Warning]` | A client connection ended without a clean close: the client process died, a timeout (`Got timeout reading communication packets`), or a proxy closed it. Some are normal; a burst at one moment is a client-side or network event. |
| `Too many connections` (error 1040) | the client | `max_connections` reached. Kolla-Ansible sets `max_connections=10000` (`galera.cnf.j2`). |
| `WSREP has not yet prepared node for application use` (error 1047) | the client | The node is not in a primary component, or not synced: it refuses queries. HAProxy should not have sent the client there. |
| `Deadlock: wsrep aborted transaction` (error 1213) | the client | Galera certification conflict: two nodes committed conflicting writes at the same time, and this one lost. |

Sources: `sql/share/errmsg-utf8.txt`, `sql/sql_parse.cc` (MariaDB 11.4.13).

## The oslo.db side, in the OpenStack services

`oslo.db` turns the MySQL client errors into its own exceptions (`exc_filters.py`), which is what you see in the
services' tracebacks:

| In the service log | Comes from | What it means |
|---|---|---|
| `DBConnectionError` with `(2002 …)`, `(2003 …)`, `(2006 …)`, `(2013 …)` or `(1047 …)` in the message | MySQL errors 2002/2003 (cannot connect), 2006 (server has gone away), 2013 (lost connection during query), 1047 (node not ready) | The service could not reach a usable database node through HAProxy. |
| `DBDeadlock` with `1213` and `Deadlock found when trying to get lock` or `Deadlock: wsrep aborted` | InnoDB deadlock, or a Galera certification conflict | Usually retried by the service (`oslo_db.api.wrap_db_retry`); a flood means many nodes take writes for the same rows at once. |
| `SQL connection failed. <n> attempts left.` | `engines.py`, WARNING | The service is starting and cannot connect; it retries. |
| `DB exceeded retry limit.` with a traceback | `api.py`, ERROR | A retried operation kept failing. |
| `DBAPIError exception wrapped.` / `DB exception wrapped.` with a traceback | `exc_filters.py`, WARNING | A database error that `oslo.db` did not classify; read the wrapped error. |

Kolla-Ansible sends all database clients through HAProxy to one active node at a time (the others are backups in the
`mariadb` backend), so a burst of `DBConnectionError` in many services at the same second is usually that node going
away and HAProxy failing over; read the HAProxy log and the database node logs at that second.

## Reading a database incident in order

1. **Status variables on every node:** which nodes are `Primary`/`Synced`, which are not.
2. **The last `Quorum results:` on each node:** `component` and `members`, with their times.
3. **What made a node leave:** `connection to peer … timed out`, `suspecting node`, a restart (`Shifting … -> CLOSED`).
4. **How it came back:** IST or SST, and whether the SST succeeded.
5. **Then the clients:** the first and last `DBConnectionError` in the services bracket the outage as they saw it.

## Validity and limits

- Every message on this page is quoted from the Galera 26.4.27, MariaDB 11.4.13 and `oslo.db` 2026.1 sources.
  **None was captured on a lab.** The exact Galera build in a Kolla 2026.1 image may be another 26.4 release, with the
  same messages or close to them. A lab request asks for real samples.
- `connection to peer … timed out` names `gmcast.peer_timeout`; the related `evs.*` timeouts that decide when a node
  is declared inactive are not detailed here.
- The SST script messages (for `mariabackup`) are not covered.
- HAProxy's `mariadb` backend layout is described from Kolla-Ansible's defaults; check `haproxy` configuration on your
  platform.

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
