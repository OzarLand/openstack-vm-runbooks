---
title: New instance in ERROR with "Exceeded maximum number of retries". The real error is on the computes that tried
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, placement]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any multinode
  network: any
  storage: any
  os: any
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, multinode (3 controllers, 2 computes)", date: 2026-10-27, note: "on a disposable lab: cause 3 (an image property both computes reject at spawn) and cause 1 (one compute broken the same way by a nova.conf change) reproduced; quick checks, disable, enable and Verification run; cause 2 not reproduced as such, the Any: lines not run" }
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#scheduler.max_attempts
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#filter_scheduler.build_failure_weight_multiplier
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#compute.consecutive_build_service_disable_threshold
  - https://docs.openstack.org/nova/2026.1/admin/scheduling.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# New instance in ERROR with "Exceeded maximum number of retries". The real error is on the computes that tried

## Symptom

A new instance goes to `ERROR`, often after a longer wait than usual, with this fault:

```text
$ openstack server show <server-id> -c status -c fault
| fault  | {'code': 500, 'message': 'Exceeded maximum number of retries. Exhausted all hosts available for retrying build failures for instance <server-id>.', ...} |
| status | ERROR |
```

This is not the scheduler's `No valid host was found` with free capacity, which comes before any compute tries. Here
the scheduler did find computes, and each one tried to build the instance and failed. The message says only that
there were no more computes to try.

## Impact

New instances fail, for one flavor, image or network, or for everyone, depending on the cause. Running instances are
not affected. A broken compute that keeps failing builds also slows every build that lands on it first, until Nova
weighs it down (see *Diagnosis*).

## Quick checks

Read-only. With admin credentials:

```bash
# 1. The fault, and when the instance was created
openstack server show <server-id> -c status -c fault -c created

# 2. The instance's actions: each compute that tried left an event, with its own error
openstack server event list <server-id>
openstack --os-compute-api-version 2.84 server event show <server-id> <request-id>
#    look for the compute_... events and their host: one per compute that tried.
#    The per-compute traceback is shown to admins only (default policy). From compute API 2.84 each failed event
#    also has a details field with the fault message; with an older microversion a non-admin sees no error text.
#    On a Kolla-Ansible 22.2.0 lab the events did not show the error: the compute that failed and rescheduled had
#    result Success, the last one no result, and none had details or a traceback. Use them for the hosts, and the
#    compute logs below for the error.

# 3. Are the computes and the agents they need up?
openstack compute service list --service nova-compute
openstack network agent list
```

On each compute named in check 2, the `nova-compute` log for the server ID has the real exception:

**Kolla-Ansible:** `sudo grep <server-id> /var/log/kolla/nova/nova-compute.log`

On the lab, with an image property both computes rejected, each compute logged:

```text
ERROR nova.compute.manager [...] [instance: <server-id>] Instance failed to spawn: nova.exception.InvalidMachineType: 'pc-i440fx-1.0' is not valid/supported machine type, Supported machine types are: [...]
```

and the conductor `WARNING nova.scheduler.utils [...] Failed to compute_task_build_instances: Exceeded maximum number
of retries. ...` then `Setting instance to ERROR state.`

**Any:** *(not run on the lab)* the `nova-compute` service log on that compute.

## Diagnosis

### How a build is retried

The scheduler returns a chosen compute and a list of alternates, `[scheduler] max_attempts` in all (Nova's default
3; **Kolla-Ansible 2026.1 writes `max_attempts = 10`** in `nova.conf`). When a compute fails to spawn the instance with
an error Nova treats as retryable, it removes the instance's allocation, and the conductor sends the build to the next
alternate. When the list is empty, the conductor raises `MaxRetriesExceeded`, a kind of `NoValidHost`, with the
message above, and the instance goes to `ERROR`.

So the fault is a summary. The cause is the error each compute hit, and it is usually the same on all of them.

Not every failure is retried. In `nova/compute/manager.py`, several abort the build at once on the first compute,
with a fault `Build of instance <server-id> aborted: …` instead of this one:

- a network allocation failure (`Failed to allocate the network(s), not rescheduling.`), including the wait for
  `vif_plugged` ([waiting for vif_plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md));
- the block device step of a boot from volume ([boot from volume](instance-fails-to-boot-from-volume.md));
- image and flavor mismatches (flavor disk or memory too small, image not active or not acceptable).

What is retried: a compute whose resource claim fails (`ComputeResourcesUnavailable`), and errors raised while
spawning the guest that are not in those lists.

### How Nova weighs a failing compute

Each compute counts its consecutive build failures, and resets the count after a success. When
`[compute] consecutive_build_service_disable_threshold` is not zero (default 10), the count is reported to the
scheduler, and the `BuildFailureWeigher` pushes computes with recent failures down by
`[filter_scheduler] build_failure_weight_multiplier` (default 1000000.0), which is meant to outweigh the other
weighers. With Nova's default weighers, a compute that fails every build is chosen last. It is not disabled: the
option's help in 2026.1 describes it as reporting build failures for the weigher.

What this means in practice:

- one broken compute among healthy ones: a few builds fail on it and are retried elsewhere, then it stops being
  chosen first. Builds succeed, slower; the broken compute stays broken;
- every compute broken the same way (a common dependency): every attempt fails, and every build ends with this fault.

### What to look for on the computes

The exception in each compute's log, or in the event traceback, names the cause. Common ones, with the runbooks that
cover them:

| The error on the compute | Go to |
|---|---|
| The resource claim failed on the compute | The compute's resources changed since the scheduler chose it, or its view disagrees with Placement: [no valid host with free capacity](no-valid-host-with-free-capacity.md), *Placement has no candidates* |
| A volume connection failure from `os_brick` while spawning | [Instance fails to boot from volume](instance-fails-to-boot-from-volume.md), cause 4 |
| No space for the instance's disk on the compute | [Compute disk full](../lifecycle/compute-disk-full-instance-paused.md), for the instance store |
| A libvirt or QEMU error when defining or starting the domain | The `nova_libvirt` logs on that compute; [restarting nova_libvirt](../lifecycle/restarting-nova-libvirt-does-not-stop-instances.md) if libvirt is stuck |

## Possible causes

### Cause 1: one broken compute

Check 2 shows one compute failing, then a later attempt on another one, or the build succeeding on retry for other
instances. The broken compute's own error tells why.

### Cause 2: a dependency shared by every compute

Every attempt fails with the same error on different computes: a storage backend unreachable from all computes,
libvirt misconfigured the same way everywhere after a change. The fault appears on every new instance.

### Cause 3: an instance request no compute can satisfy at spawn time

The scheduler's view (Placement, filters) said yes, but each compute fails on something the scheduler does not check,
for example an image property or a device the hosts cannot provide. Every attempt fails with the same error, for this
flavor or image only.

## Resolution

Fix the cause each compute reported, using the runbook it points to. Then build again: an instance in `ERROR` from a
failed build is not retried by Nova.

To keep new builds off a compute while it is being repaired:

> ⚠️ **State-changing.** Disables the compute for scheduling: no new instance lands on it; running instances are not
> affected. Undo: `openstack compute service set --enable <host> nova-compute`.

```bash
openstack compute service set --disable --disable-reason "<why>" <host> nova-compute
```

Once a compute is repaired, a successful build on it resets its failure count, and the weigher stops pushing it down.

## Verification

```bash
openstack server create --image <image> --flavor <flavor> --network <network> --wait <test-name>
openstack server show <test-name> -c status -c OS-EXT-SRV-ATTR:host
# expected: ACTIVE, on a compute that was failing before if it was repaired and enabled
openstack server delete --wait <test-name>
```

> ⚠️ **State-changing.** Creates a test instance, deleted at the end. Needs quota, an image, a flavor and a network.

## Lab observations

On a Kolla-Ansible 22.2.0 lab (2026.1, 3 controllers, 2 computes, `max_attempts = 10` in the `nova-scheduler` and
`nova-conductor` `nova.conf`):

- Cause 3: an image with `hw_machine_type=pc-i440fx-1.0`, which neither compute supports. The build tried compute-1,
  then compute-2, and ended in `ERROR` with the fault above after 34 s. Two attempts only: with two computes there is
  one alternate, whatever `max_attempts` says.
- Cause 1: the same machine type set in `[libvirt] hw_machine_type` of compute-1 only. Of four builds in a row, the
  first tried compute-1, failed and was retried on compute-2 (37 s); the next three went straight to compute-2
  (20 to 22 s). All four `ACTIVE`.
- `compute service set --disable` and `--enable` behaved as written. After the repair (which restarted
  `nova_compute`) and `--enable`, the Verification instance was built on compute-1.

## Validity and limits

- Written from Nova 2026.1 (`nova/conductor/manager.py`, `nova/compute/manager.py`, `nova/exception.py`, the
  `max_attempts`, `consecutive_build_service_disable_threshold` and `build_failure_weight_multiplier` options) and
  Kolla-Ansible 2026.1 (`max_attempts = 10` in the `nova` and `nova-cell` `nova.conf` templates). Causes 1 and 3
  reproduced on a lab with two computes (see *Lab observations*); cause 2 not reproduced as such (cause 3 shows the
  same picture: every compute fails with the same error). The weigher was seen only through where builds landed.
- The option `consecutive_build_service_disable_threshold` has "disable" in its name, but its 2026.1 help describes
  only reporting failures for the weigher. On another release, read that release's help.
- Not covered: retries of move operations (resize, cold migration), Ironic, and custom weigher configurations.

## Related

- [No valid host was found, although capacity exists](no-valid-host-with-free-capacity.md): when the scheduler
  finds no compute at all.
- [Instance fails to boot from volume](instance-fails-to-boot-from-volume.md)
- [What a VM creation traverses, and how to trace one failed boot by request id](../diagnostics/what-a-vm-creation-traverses-and-tracing-by-request-id.md)
- Nova 2026.1: [compute schedulers](https://docs.openstack.org/nova/2026.1/admin/scheduling.html),
  [configuration reference](https://docs.openstack.org/nova/2026.1/configuration/config.html)

---

*Provenance: written by agent:ozarland-author, from the Nova and Kolla-Ansible 2026.1 documentation and source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=scheduling/build-fails-exceeded-maximum-number-of-retries.md&title=Feedback%3A+scheduling/build-fails-exceeded-maximum-number-of-retries.md):
your release, your deployment method, and what happened.
