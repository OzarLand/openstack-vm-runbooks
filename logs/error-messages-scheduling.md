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
Nova `stable/2026.1`; `<…>` marks a value. The *Lab observations* section shows the lines captured on a Kolla-Ansible
2026.1 lab.

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
    Placement (resources, traits, aggregates, or computes not registered); see
    [reading the Placement logs](reading-placement-logs.md) and [reading the Nova logs](reading-nova-logs.md);
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
    [`conf/scheduler.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/conf/scheduler.py); Kolla-Ansible
    2026.1 renders `max_attempts = 10` in `/etc/kolla/nova-scheduler/nova.conf`).
- **Read next:** the **computes that tried**, not the scheduler. Each failed attempt is logged on its compute as
  `Build of instance <uuid> was re-scheduled: <reason>` (`RescheduledException`), with the real error just before.
  Find them with `openstack server event show <server-id> <request-id>` (the event's `host` names the compute; see
  the runbook for the microversion and the admin-only details), then `nova-compute.log` on each. On the lab the
  `compute__do_build_and_run_instance` event of a rescheduled build read `result: Success` with `traceback: null`:
  the error is in `nova-compute.log`, not in the event. On Kolla-Ansible 2026.1 the `was re-scheduled` text did not
  appear as a line of its own but as the exception appended to the next WARNING line of the request
  (`...: nova.exception.RescheduledException: Build of instance <uuid> was re-scheduled: <reason>`); `grep
  're-scheduled'` finds it. Where the retry record carries it, the **conductor** logs `Error from last host: <host>
  (node <node>): <exception>` (ERROR, logger `nova.scheduler.utils`, in `nova-conductor.log`).
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

## Lab observations

Kolla-Ansible 22.2.0 (OpenStack 2026.1) all-in-one, one compute, ML2/OVS, logs at INFO, 2026-10-27. Real lines,
redacted:

- A flavor with more RAM than the host: fault `No valid host was found. ` (empty reason) 4 s after the boot;
  `nova-scheduler.log`:

  ```text
  2026-10-27 07:03:41.655 94 INFO nova.scheduler.manager [None req-<request-id> <user-id> <project-id> - - default default] Got no allocation candidates from the Placement API. This could be due to insufficient resources or a temporary occurrence as compute nodes start up.
  ```

  and `nova-conductor.log`:

  ```text
  2026-10-27 07:03:42.081 88 ERROR nova.conductor.manager [None req-<request-id> ...] Failed to schedule instances: nova.exception_Remote.NoValidHost_Remote: No valid host was found.
  2026-10-27 07:03:42.200 88 WARNING nova.scheduler.utils [None req-<request-id> ...] Failed to compute_task_build_instances: No valid host was found.
  2026-10-27 07:03:42.202 88 WARNING nova.scheduler.utils [None req-<request-id> ...] [instance: <server-id>] Setting instance to ERROR state.: nova.exception_Remote.NoValidHost_Remote: No valid host was found.
  ```

- A flavor with the extra spec `capabilities:hypervisor_type=s390-none`: fault `No valid host was found. There are
  not enough hosts available.`; `nova-scheduler.log`:

  ```text
  2026-10-27 07:03:49.795 101 INFO nova.filters [None req-<request-id> ...] Filter ComputeCapabilitiesFilter returned 0 hosts
  2026-10-27 07:03:49.802 101 INFO nova.filters [None req-<request-id> ...] Filtering removed all hosts for the request with instance ID '<server-id>'. Filter results: ['ComputeFilter: (start: 1, end: 1)', 'ComputeCapabilitiesFilter: (start: 1, end: 0)']
  ```

- An image with `hw_machine_type=pc-q35-nonexistent` (a reschedulable failure on the only compute): fault `Exceeded
  maximum number of retries. Exhausted all hosts available for retrying build failures for instance <uuid>.` 18 s
  after the boot; nothing in `nova-scheduler.log` at INFO; `nova-compute.log`, then `nova-conductor.log`:

  ```text
  2026-10-27 07:04:36.725 6 ERROR nova.compute.manager [None req-<request-id> ...] [instance: <server-id>] Failed to build and run instance: nova.exception.InvalidMachineType: 'pc-q35-nonexistent' is not valid/supported machine type, Supported machine types are: [...]
  2026-10-27 07:04:36.735 6 WARNING neutronclient.v2_0.client [None req-<request-id> ...] The python binding code in neutronclient is deprecated in favor of OpenstackSDK, please use that as this will be removed in a future release.: nova.exception.RescheduledException: Build of instance <server-id> was re-scheduled: 'pc-q35-nonexistent' is not valid/supported machine type, [...]
  2026-10-27 07:04:39.923 89 ERROR nova.scheduler.utils [None req-<request-id> ...] [instance: <server-id>] Error from last host: compute-1 (node compute-1): ['Traceback (most recent call last):\n', [...]
  2026-10-27 07:04:39.927 89 WARNING nova.scheduler.utils [None req-<request-id> ...] [instance: <server-id>] Setting instance to ERROR state.: nova.exception.MaxRetriesExceeded: Exceeded maximum number of retries. Exhausted all hosts available for retrying build failures for instance <server-id>.
  ```

- Not observed: the `Exceeded max scheduling attempts <n>` reason, the DEBUG lines, a reschedule to an alternate
  compute (one compute only).

## Validity and limits

- Written from Nova `stable/2026.1` source; the messages in the sections above are format strings from that source,
  and the fault message shown by the API can be shortened.
- Lab-checked on a Kolla-Ansible 2026.1 all-in-one (one compute, INFO logs): the no-allocation-candidates, filter and
  `Exhausted all hosts` cases, with the lines in *Lab observations*. Not covered: `Exceeded max scheduling attempts`,
  reschedules across several computes, DEBUG output, other deployment methods.
- The cells v2 details (`cell0`, super conductor) depend on the deployment's cell layout.

---

*Provenance: written by agent:ozarland-author, from upstream source code (Nova) and a lab run on Kolla-Ansible 2026.1
(2026-10-27, by the lab reviewer). Reviewed in its pull request (see the
page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs/error-messages-scheduling.md&title=Feedback%3A+logs/error-messages-scheduling.md):
your release, your deployment method, and what happened.
