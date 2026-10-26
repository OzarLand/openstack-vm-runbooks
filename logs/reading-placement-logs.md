---
title: Reading the Placement logs, and the Placement lines in Nova's logs
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [placement, nova]
deployment_methods: [kolla-ansible, any]
environment:
  topology: multinode
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by: []
upstream_refs:
  - https://opendev.org/openstack/placement/src/branch/stable/2026.1/placement/requestlog.py
  - https://opendev.org/openstack/placement/src/branch/stable/2026.1/placement/objects/allocation.py
  - https://opendev.org/openstack/placement/src/branch/stable/2026.1/placement/objects/resource_provider.py
  - https://opendev.org/openstack/placement/src/branch/stable/2026.1/placement/fault_wrap.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/scheduler/client/report.py
  - https://opendev.org/openstack/nova/src/branch/stable/2026.1/nova/scheduler/manager.py
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Reading the Placement logs, and the Placement lines in Nova's logs

Placement holds the inventory of every compute and the allocation of every instance. Its log answers two questions:
did Nova's request reach Placement, and why did Placement refuse it. Nova's side of each exchange is in
`nova-scheduler.log` (allocation candidates, claims) and `nova-compute.log` (inventory updates, allocation moves),
and most of Nova's error lines carry Placement's request id, which links the two. Messages are quoted from Placement
and Nova 2026.1.

Fixing allocations that are out of sync is a runbook; this page reads the lines.

## Where the log is

**Kolla-Ansible:** `/var/log/kolla/placement/placement-api.log` (`log_file = /var/log/kolla/placement/<service>.log`
in `placement.conf.j2`), and the uWSGI request log `placement-api-uwsgi.log`, on each controller.

**Any:** `/var/log/placement/` or the journal, depending on the packaging.

## The request line

Placement logs one line per request at `INFO` (`requestlog.py`):

```text
<client address> "<METHOD> <URI>" status: <code> len: <bytes> microversion: <version>
```

With the `oslo.log` prefix, it carries Placement's own request id and, for calls from Nova, Nova's request id as the
global request id (Nova's report client passes `global_request_id=context.global_id`). So the request line of an
allocation write can be found from the instance's create request:

```bash
# Kolla-Ansible, on the controllers
grep "<nova-request-id>" /var/log/kolla/placement/placement-api.log
grep -E 'status: (409|400|500)' /var/log/kolla/placement/placement-api.log | tail -20
```

The useful URIs: `GET /allocation_candidates?...` (the scheduler asks where the request fits), `PUT /allocations/<consumer>`
(a claim; the consumer is the instance UUID, or a migration UUID during a move),
`PUT /resource_providers/<uuid>/inventories` (a compute reports its capacity), `DELETE /allocations/<consumer>`.

## Status codes, and what Placement logs with them

| Status on the request line | Placement's own line (message text) | Level | What it means |
|---|---|---|---|
| 409 on `PUT /allocations/...` | `Over capacity for <rc> on resource provider <rp>. Needed: <n>, Used: <n>, Capacity: <n>` | WARNING | The claim does not fit: `Capacity` is `(total - reserved) * allocation_ratio`. Another instance took the room between scheduling and claim, or the inventory changed. |
| 409 (or 400) on `PUT /allocations/...` | `Allocation for <rc> on resource provider <rp> violates min_unit, max_unit, or step_size. Requested: <n>, min_unit: <n>, max_unit: <n>, step_size: <n>` | WARNING | The flavor asks for more of one resource than a single provider allows in one allocation (`max_unit`). |
| 409 on allocations or inventories | `Exceeded retry limit of <n> on allocations write` | WARNING | Concurrent writes to the same providers kept conflicting (`allocation_conflict_retry_count`, default 10). |
| 200 on `PUT .../inventories` | `Resource provider <uuid> is now over-capacity for <rc>` | WARNING | A compute reported less capacity than is already allocated (allocation ratio lowered, reserved raised, disk shrank). Existing instances stay; new claims on that resource fail. |
| 409 on allocations | — (in Nova: `consumer generation conflict`) | — | Two writers updated the same consumer's allocations; the second must re-read. Nova retries some of these. |
| 500 | `Placement API unexpected error: <exception>` with a traceback | ERROR | An unhandled exception, often the database underneath. |

Sources: `placement/objects/allocation.py`, `placement/objects/resource_provider.py`, `placement/fault_wrap.py`,
`placement/conf/placement.py`.

The client (Nova) receives the matching error text in the response body, for example `Unable to create allocation
for '<rc>' on resource provider '<rp>'. The requested amount would exceed the capacity.` or `… would violate
inventory constraints.` (`placement/exception.py`).

## Placement in Nova's logs

### nova-scheduler

| Line (message text) | Level | What it means |
|---|---|---|
| `Got no allocation candidates from the Placement API. This could be due to insufficient resources or a temporary occurrence as compute nodes start up.` | INFO | `GET /allocation_candidates` returned nothing: no provider has the resources and traits requested. Check the request line in Placement's log for the query. |

Source: `nova/scheduler/manager.py`. What to do next: [No valid host with free capacity](../scheduling/no-valid-host-with-free-capacity.md).

### nova-compute (the report client)

The compute's resource tracker talks to Placement through Nova's report client (`nova/scheduler/client/report.py`).
Its error lines start with **Placement's request id in brackets**, which you can grep in `placement-api.log`:

| Line (message text) | Level | What it means |
|---|---|---|
| `[<placement-req-id>] Failed to update inventory to [<inventory>] for resource provider with UUID <uuid>.  Got <status>: <error text>` | ERROR | Placement refused the compute's inventory. With 409 and allocations in the way, Nova raises `InventoryInUse`. |
| `[<placement-req-id>] Failed to retrieve resource provider tree from placement API for UUID <uuid>. Got <status>: <error text>.` and the other `[<id>] Failed to retrieve …` lines | ERROR | A read from Placement failed. |
| `[<placement-req-id>] Created resource provider record via placement API for resource provider with UUID <uuid> and name <name>.` | INFO | The compute registered itself (first start, or after its provider was deleted). |
| `Failed to save allocation for <consumer>. Got HTTP <status>: <text>` | WARNING | An allocation write from the compute side (for example during a move) was refused. |
| `Unable to post allocations for consumer <uuid> (<status> <text>)` / `Unable to delete allocation for <type> <uuid>: (<status> <text>)` | WARNING or ERROR | An allocation could not be written or removed; leftovers become phantom usage. |
| `Placement API service is not responding.` | WARNING (ERROR at client start) | Nova cannot connect to Placement: the inventory is not refreshed while it lasts. |
| `Placement service credentials do not work.` | WARNING (ERROR at client start) | The `[placement]` credentials in `nova.conf` are refused by Keystone. |
| `The placement API endpoint was not found.` | ERROR | No Placement endpoint in the catalog for the configured interface and region. |

Sources: `nova/scheduler/client/report.py`.

```bash
# On a compute: Placement problems seen by its report client
grep -nE "Placement API service is not responding|Placement service credentials|Failed to update inventory|Failed to save allocation" \
  /var/log/kolla/nova/nova-compute.log | tail
# Then follow one placement request id to Placement
grep "<placement-req-id>" /var/log/kolla/placement/placement-api.log
```

## Reading a Placement problem in order

1. **Nova's line first**: the scheduler (`Got no allocation candidates`) or the compute (`Failed to …` with a request
   id in brackets).
2. **Placement's request line** for that id: method, URI, status.
3. **Placement's WARNING next to it**: over capacity, unit constraints, retry limit.
4. **The numbers**: `openstack resource provider inventory list <uuid>` and
   `openstack resource provider usage show <uuid>` (with `osc-placement`) against what the line says.

## Validity and limits

- Every message is quoted from Placement and Nova 2026.1. **None was captured on a lab.** A lab request asks for real
  samples.
- Which status code accompanies each Placement warning follows from the exceptions raised in the source
  (`InvalidAllocationCapacityExceeded`, `InvalidAllocationConstraintsViolated`, `ResourceProviderConcurrentUpdateDetected`);
  the exact code was not observed.
- The `osc-placement` commands require that client plugin.

## Related

- [No valid host with free capacity](../scheduling/no-valid-host-with-free-capacity.md)
- [Stale records, phantom allocations](../lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md)
- [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md)
- [Logs index](README.md)

---

*Provenance: written by agent:ozarland-author, from Placement and Nova 2026.1 source code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Freading-placement-logs.md&title=Feedback%3A+logs%2Freading-placement-logs.md):
your release, your deployment method, and what happened.
