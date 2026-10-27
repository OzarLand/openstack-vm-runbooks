---
title: Reading the Octavia logs. Health manager, amphora heartbeats, failovers and the worker
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [octavia, nova, neutron]
deployment_methods: [kolla-ansible, any]
environment:
  topology: multinode, amphora provider
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by: []
upstream_refs:
  - https://opendev.org/openstack/octavia/src/branch/stable/2026.1/octavia/controller/healthmanager/health_manager.py
  - https://opendev.org/openstack/octavia/src/branch/stable/2026.1/octavia/amphorae/drivers/health/heartbeat_udp.py
  - https://opendev.org/openstack/octavia/src/branch/stable/2026.1/octavia/amphorae/drivers/haproxy/rest_api_driver.py
  - https://opendev.org/openstack/octavia/src/branch/stable/2026.1/octavia/common/config.py
  - https://opendev.org/openstack/octavia/src/branch/stable/2026.1/octavia/controller/worker/v2/flows/amphora_flows.py
  - https://opendev.org/openstack/octavia/src/branch/stable/2026.1/octavia/controller/worker/v2/tasks/amphora_driver_tasks.py
  - https://opendev.org/openstack/octavia/src/branch/stable/2026.1/octavia/controller/worker/v2/controller_worker.py
  - https://opendev.org/openstack/octavia/src/branch/stable/2026.1/octavia/db/repositories.py
  - https://opendev.org/openstack/octavia/src/branch/stable/2026.1/octavia/controller/housekeeping/house_keeping.py
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Reading the Octavia logs. Health manager, amphora heartbeats, failovers and the worker

With the amphora provider, each load balancer runs in one or two service VMs (amphorae) that send a heartbeat to the
Octavia health manager over the management network. When heartbeats stop arriving, the health manager fails the
amphora over: it builds a new one, through Nova and Neutron. The logs say whether amphorae were really down, or
whether the controllers stopped hearing them. Messages are quoted from Octavia 2026.1.

## Where the logs are

**Kolla-Ansible:** `/var/log/kolla/octavia/` (`log_dir` in `octavia.conf.j2`): `octavia-api.log`,
`octavia-worker.log`, `octavia-health-manager.log`, `octavia-housekeeping.log`, `octavia-driver-agent.log`, from the
containers `octavia_api`, `octavia_worker`, `octavia_health_manager`, `octavia_housekeeping`, `octavia_driver_agent`.

**Any:** `/var/log/octavia/` or the journal. The amphora's own logs (its agent, HAProxy, keepalived) are inside the
amphora VM.

## Heartbeats and failover: the health manager

An amphora sends a heartbeat every `heartbeat_interval` seconds (default 10); the health manager checks every
`health_check_interval` seconds (default 3) for amphorae whose last heartbeat is older than `heartbeat_timeout`
(default 60), and fails them over (`[health_manager]` options in `common/config.py`).

| Line (message text) | Level | What it means | Next |
|---|---|---|---|
| `Stale amphora's id is: <amphora-id>` | INFO | No heartbeat for `heartbeat_timeout`: a failover of this amphora starts. | the worker log for the failover |
| `Attempted <n> failovers of amphora`, `Failed at <n> …`, `Cancelled <n> …`, `Successfully completed <n> failovers of amphora` | INFO | Summary of a batch. **Many failovers at once** usually means the controllers stopped receiving heartbeats (management network, health manager overload), not that many amphorae died together. | the management network; `failover_threshold` |
| `Stale amphora count reached the threshold (<n>). <n> amphorae were set into FAILOVER_STOPPED status.` | ERROR | `[health_manager] failover_threshold` is set and that many amphorae went stale at once: instead of failing them over, Octavia stops (`db/repositories.py`). The protection against a mass failover when the controllers lose the management network. | the management network; then bring the amphorae back once heartbeats are seen again |
| `Amphora <amphora-id> health message was processed too slowly: <n>s! The system may be overloaded or otherwise malfunctioning. This heartbeat has been ignored and no update was made to the amphora health entry. THIS IS NOT GOOD.` | WARNING | The health manager is too slow: heartbeats are dropped although they arrive. If it lasts, healthy amphorae are declared stale and failed over. | CPU of the health manager; its database |
| `Health Manager experienced an exception processing a heartbeat message from <address>. Ignoring this packet. Exception: <error>` | WARNING | A heartbeat could not be decoded: wrong `heartbeat_key` (the HMAC key), or a packet that is not a heartbeat. | `[health_manager] heartbeat_key` on controllers and in amphorae |
| `Amphora <amphora-id> health message reports <n> listeners when <n> expected` | WARNING | The amphora runs a different number of listeners than the database expects (a configuration push did not complete). The heartbeat is then **not** recorded: if the mismatch lasts, the amphora goes stale and is failed over although it still sends heartbeats. | the worker log for that load balancer |
| `Load balancer <id> is in immutable state <state>. Skipping failover.` | WARNING | A failover was due but the load balancer is in a `PENDING_*` state: something else is changing it, or a previous operation is stuck. The health check then stops for this cycle: other stale amphorae wait for the next one (`health_check` in `health_manager.py`). | the worker log; the load balancer's provisioning status |

Sources: `controller/healthmanager/health_manager.py`, `amphorae/drivers/health/heartbeat_udp.py`,
`db/repositories.py`.

```bash
# Kolla-Ansible, on the controllers
grep -cE "Stale amphora's id is" /var/log/kolla/octavia/octavia-health-manager.log
grep -E "processed too slowly|exception processing a heartbeat" /var/log/kolla/octavia/octavia-health-manager.log | tail
```

## The worker: talking to the amphora

The worker configures an amphora through the amphora agent's REST API (`amphorae/drivers/haproxy/rest_api_driver.py`).
What it logs when the amphora does not answer depends on the operation.

**A new amphora (create, failover).** The build and failover flows wait for the new amphora with
`AmphoraComputeConnectivityWait` inside a retry subflow (`_retry_flow` in `controller/worker/v2/flows/amphora_flows.py`,
with `raise_retry_exception=True`):

| Line (message text) | Level | What it means |
|---|---|---|
| `Could not connect to instance. Retrying.` | WARNING | One attempt failed: normal for a few minutes while a new amphora boots. Each attempt raises `AmpConnectionRetry`, and the `AmpRetry` controller runs the task again, up to `[haproxy_amphora] connection_max_retries` (120) times, `connection_retry_interval` (5 s) apart. Each attempt also waits for the request timeout (`rest_request_conn_timeout`, 10 s), so 120 × 5 s is a lower bound. |
| `Could not connect to amphora, exception caught: <error>` | the exception text, in the worker's failure line and traceback (for a failover: `Amphora <id> failover exception: …` or `LB <id> failover exception: …`, ERROR, `controller_worker.py`) | The retries are used up and the flow reverts: the amphora never answered. It did not boot (Nova), has no management network address (Neutron), or the controllers cannot route to the management network. The create or failover fails. |

On this path the worker does **not** log `Connection retries … exhausted`: grep for `Could not connect to`.

**An existing amphora (a configuration push, a listener update).** Calls without the retry flag loop inside the
driver, then give up with:

| Line (message text) | Level | What it means |
|---|---|---|
| `Connection retries (currently set to <n>) exhausted.  The amphora is unavailable. Reason: <error>` | ERROR | An amphora that used to answer is unreachable during a configuration call: it died, or the management network is cut. |

When an amphora build fails in Nova or Neutron, the worker's traceback names the Nova or Neutron error; continue
with [Nova's logs](reading-nova-logs.md) for the amphora's server id.

## Reading an Octavia problem in order

1. **Health manager**: stale amphorae and failover batches, with times. Many at once points at the controllers or
   the management network.
2. **Processing delays and decode errors**: `processed too slowly`, `exception processing a heartbeat`.
3. **Worker**: `Could not connect to` (a new amphora that never answered), `Connection retries … exhausted` (an
   existing amphora unreachable), Nova and Neutron errors during the new amphora's build.
4. **Load balancer state**: `openstack loadbalancer show <lb>` and `openstack loadbalancer amphora list`.

## Validity and limits

- Every message is quoted from Octavia 2026.1. **None was captured on a lab.**
- The amphora provider only; OVN provider load balancers have no amphora and are not covered.
- The amphora's internal logs are not covered.
- `octavia-housekeeping.log` holds routine lines only: `Purged db record for Amphora ID: <id>`, `Deleted load balancer
  id : <id>` (old deleted records removed after `amphora_expiry_age` / `load_balancer_expiry_age`), `Rotated
  certificates for <n> amphora` (`controller/housekeeping/house_keeping.py`). Its error lines are not covered.

## Related

- [Reading the Nova logs](reading-nova-logs.md)
- [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md)
- [Logs index](README.md)

---

*Provenance: written by agent:ozarland-author, from Octavia 2026.1 source code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Freading-octavia-logs.md&title=Feedback%3A+logs%2Freading-octavia-logs.md):
your release, your deployment method, and what happened.
