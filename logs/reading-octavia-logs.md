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
authored_by: agent:ozarland-author
last_verified: 2026-10-26
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
| `Amphora <amphora-id> health message was processed too slowly: <n>s! The system may be overloaded or otherwise malfunctioning. This heartbeat has been ignored and no update was made to the amphora health entry. THIS IS NOT GOOD.` | WARNING | The health manager is too slow: heartbeats are dropped although they arrive. If it lasts, healthy amphorae are declared stale and failed over. | CPU of the health manager; its database |
| `Health Manager experienced an exception processing a heartbeat message from <address>. Ignoring this packet. Exception: <error>` | WARNING | A heartbeat could not be decoded: wrong `heartbeat_key` (the HMAC key), or a packet that is not a heartbeat. | `[health_manager] heartbeat_key` on controllers and in amphorae |
| `Amphora <amphora-id> health message reports <n> listeners when <n> expected` | WARNING | The amphora runs a different number of listeners than the database expects (a configuration push did not complete). | the worker log for that load balancer |
| `Load balancer <id> is in immutable state <state>. Skipping failover.` | WARNING | A failover was due but the load balancer is in a `PENDING_*` state: something else is changing it, or a previous operation is stuck. | the worker log; the load balancer's provisioning status |

Sources: `controller/healthmanager/health_manager.py`, `amphorae/drivers/health/heartbeat_udp.py`.

```bash
# Kolla-Ansible, on the controllers
grep -cE "Stale amphora's id is" /var/log/kolla/octavia/octavia-health-manager.log
grep -E "processed too slowly|exception processing a heartbeat" /var/log/kolla/octavia/octavia-health-manager.log | tail
```

## The worker: talking to the amphora

The worker configures an amphora through the amphora agent's REST API (`amphorae/drivers/haproxy/rest_api_driver.py`):

| Line (message text) | Level | What it means |
|---|---|---|
| `Could not connect to instance. Retrying.` | WARNING | The worker cannot reach the amphora agent (yet): normal for a few minutes while a new amphora boots; it retries every `connection_retry_interval` (5 s) up to `connection_max_retries` (120). |
| `Connection retries (currently set to <n>) exhausted.  The amphora is unavailable. Reason: <error>` | ERROR | The amphora never answered: it did not boot (Nova), has no management network address (Neutron), or the controllers cannot route to the management network. The create or failover fails. |

When an amphora build fails in Nova or Neutron, the worker's traceback names the Nova or Neutron error; continue
with Nova's logs for the amphora's server id (the Nova page of this folder).

## Reading an Octavia problem in order

1. **Health manager**: stale amphorae and failover batches, with times. Many at once points at the controllers or
   the management network.
2. **Processing delays and decode errors**: `processed too slowly`, `exception processing a heartbeat`.
3. **Worker**: connection retries exhausted, Nova and Neutron errors during the new amphora's build.
4. **Load balancer state**: `openstack loadbalancer show <lb>` and `openstack loadbalancer amphora list`.

## Validity and limits

- Every message is quoted from Octavia 2026.1. **None was captured on a lab.**
- The amphora provider only; OVN provider load balancers have no amphora and are not covered.
- The amphora's internal logs are not covered.

## Related

- [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md)
- [Logs index](README.md)

---

*Provenance: written by agent:ozarland-author, from Octavia 2026.1 source code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Freading-octavia-logs.md&title=Feedback%3A+logs%2Freading-octavia-logs.md):
your release, your deployment method, and what happened.
