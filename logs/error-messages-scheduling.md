---
title: "Error messages: scheduling. No valid host was found, Exceeded maximum number of retries"
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [nova, placement]
deployment_methods: [any]
environment:
  topology: any
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by: []
upstream_refs:
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/exception.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/scheduler/manager.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/filters.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/scheduler/utils.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/conductor/manager.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/conf/scheduler.py
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Error messages: scheduling. No valid host was found, Exceeded maximum number of retries

This page belongs to the dictionary of frequent OpenStack error messages. For each message: where it appears, what it
means, the usual causes, the next log to read, and the runbook that fixes it. The messages are the format strings of
Nova `stable/2026.1`, not captured lines; `<…>` marks a value.

The two messages are related: `MaxRetriesExceeded` is a subclass of `NoValidHost` in
[`nova/exception.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/exception.py), but they point to
different places. `No valid host` is decided by the scheduler before any compute is tried; `Exceeded maximum number of
retries` comes after one or more computes tried and failed.

## `No valid host was found. <reason>`

- **Where:** in the instance's fault (`openstack server show <server-id> -c fault`), with the instance in `ERROR`;
  in `nova-conductor.log` as `Failed to schedule instances` with the traceback; the instance is then recorded in
  `cell0` (`_bury_in_cell0` in
  [`conductor/manager.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/conductor/manager.py)), so it
  never reached a compute.
- **Means:** the scheduler found no host for the request. Two reasons appear in the source:
  - `There are not enough hosts available.` (`scheduler/manager.py`): Placement returned candidates, but after the
    filters, weighers and claims fewer hosts were left than instances requested; at DEBUG the scheduler logs
    `There are <n> hosts available but <m> instances requested to build.`;
  - an empty reason, after `Got no allocation candidates from the Placement API. This could be due to insufficient
    resources or a temporary occurrence as compute nodes start up.` (INFO, `scheduler/manager.py`): Placement itself
    returned nothing.
- **Read next:** `nova-scheduler.log` for the request id of the failed boot (`openstack server event list
  <server-id>`). Look for:
  - `Got no allocation candidates from the Placement API` → the request does not fit any resource provider in
    Placement (resources, traits, aggregates, or computes not registered);
  - `Filter <name> returned 0 hosts` (INFO) → that filter removed the last hosts. The INFO line `Filtering removed all
    hosts for the request with instance ID '<uuid>'. Filter results: <results>` lists which filters ran and how many
    hosts each left (the DEBUG version lists the hosts) ([`nova/filters.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/filters.py)).
- **Usual causes:** not enough free resources for the flavor; computes disabled, down or not mapped to a cell; an
  image or flavor property that no host satisfies (traits, aggregates, availability zone, NUMA, PCI); allocation
  ratios or reserved resources; Placement out of sync with Nova.
- **Fix:** [No valid host was found, although capacity exists](../scheduling/no-valid-host-with-free-capacity.md).

## `Exceeded maximum number of retries. <reason>`

- **Where:** in the instance's fault, with the instance in `ERROR`, after a build was rescheduled.
- **Means:** at least one compute tried to build the instance and failed with a reschedulable error, and no alternate
  host was left. The reason, from `conductor/manager.py` and `scheduler/utils.py`, is one of:
  - `Exhausted all hosts available for retrying build failures for instance <uuid>.` (the alternates returned by the
    scheduler were all used);
  - `Exceeded max scheduling attempts <n> for instance <uuid>. Last exception: <exception>` (the count of attempts
    passed `[scheduler] max_attempts`, 3 by default, in
    [`conf/scheduler.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/conf/scheduler.py)).
- **Read next:** the **computes that tried**, not the scheduler. Each failed attempt is logged on its compute as
  `Build of instance <uuid> was re-scheduled: <reason>` (`RescheduledException`), with the real error just before.
  Find them with `openstack server event show <server-id> <request-id>` (the per-compute tracebacks; see the runbook
  for the microversion and the admin-only details), then `nova-compute.log` on each. Where the retry record carries
  it, the scheduler also logs `Error from last host: <host> (node <node>): <exception>` (ERROR,
  `scheduler/utils.py`).
- **Usual causes:** the same failure on every compute: port binding or VIF plugging, image download, volume attach,
  libvirt refusing the domain (CPU model, missing feature), disk space. These have their own pages in this dictionary.
- **Fix:** [Exceeded maximum number of retries. The real error is on the computes that tried](../scheduling/build-fails-exceeded-maximum-number-of-retries.md).

## Related lines

| Line (format) | Where | Meaning |
|---|---|---|
| `Build of instance <uuid> aborted: <reason>` (`BuildAbortException`) | `nova-compute.log` | The compute failed the build without rescheduling: the instance goes to `ERROR` on that compute, with this reason as fault |
| `Failed to <service>_<method>: <exception>` and `Setting instance to <state> state.` | `nova-conductor.log` (WARNING, `scheduler/utils.py`) | The conductor records the failure and sets the instance state |

## A short reading order

1. `openstack server show <server-id> -c fault -c OS-EXT-SRV-ATTR:host`: `No valid host` with no host, or `Exceeded
   maximum number of retries`?
2. `No valid host`: `nova-scheduler.log` for the request id: no allocation candidates, or which filter returned 0
   hosts.
3. `Exceeded maximum number of retries`: `openstack server event show` for the computes that tried, then
   `nova-compute.log` on each for `was re-scheduled` and the error before it.

## Validity and limits

- Written from Nova `stable/2026.1` source only. Every message is a format string from that source, not a captured
  line; the fault message shown by the API can be shortened.
- Not run on a lab yet (the No valid host runbook linked above is lab-verified; this page is not). A capture of the
  scheduler's `Filter ... returned 0 hosts` and `Filtering removed all hosts` lines for a flavor that no host
  satisfies is requested.
- The cells v2 details (`cell0`, super conductor) depend on the deployment's cell layout.

---

*Provenance: written by agent:ozarland-author, from upstream source code (Nova). Reviewed in its pull request (see the
page history). Checked by a human operator: not yet.*
