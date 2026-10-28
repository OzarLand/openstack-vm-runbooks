---
title: Placement allocations do not match the instances. Compute looks full or empty, audit and heal
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
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, multinode (3 controllers, 2 computes)", date: 2026-10-27, note: "the three causes injected by hand with osc-placement on a disposable lab (orphan consumer, stale second provider, deleted allocation); quick checks, saving, audit, audit --delete, allocation unset, heal_allocations (one instance, --force, delete then heal) and Verification run; the undo from the saved JSON, heal of every instance, No valid host and a real evacuation not run" }
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/admin/troubleshooting/orphaned-allocations.html
  - https://docs.openstack.org/nova/2026.1/cli/nova-manage.html#placement-audit
  - https://docs.openstack.org/nova/2026.1/cli/nova-manage.html#placement-heal-allocations
  - https://docs.openstack.org/osc-placement/latest/
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Placement allocations do not match the instances. Compute looks full or empty, audit and heal

Placement holds one allocation per consumer (an instance, or a migration) on each resource provider (a compute
node). The scheduler trusts these allocations, not the hypervisor. When they drift from what really runs, a compute
looks fuller than it is (leftover allocations of deleted or moved instances) or emptier than it is (instances with
no allocation, which the scheduler then overcommits).

## Symptom

One or more of these, usually after an evacuation, a failed migration or resize, a compute rebuilt or renamed, or a
Placement database restored from a backup:

- The usage Placement reports for a compute is higher, or lower, than the sum of the flavors of the instances that
  run on it:

  ```text
  $ openstack resource provider usage show <rp-uuid>
  +----------------+-------+
  | resource_class | usage |
  +----------------+-------+
  | VCPU           |    48 |
  | MEMORY_MB      | 98304 |
  | DISK_GB        |   480 |
  +----------------+-------+
  ```

- New instances fail with `No valid host was found` while the computes have room, and the scheduler log says:

  ```text
  INFO nova.scheduler.manager [...] Got no allocation candidates from the Placement API. This could be due to insufficient resources or a temporary occurrence as compute nodes start up.
  ```

- The `nova-compute` log of a compute repeats, at each resource update, one of these lines from
  `nova/compute/resource_tracker.py`:

  ```text
  INFO nova.compute.resource_tracker [...] Instance <consumer-uuid> has allocations against this compute host but is not found in the database.
  WARNING nova.compute.resource_tracker [...] Instance <server-id> has been moved to another host <host>(<node>). There are allocations remaining against the source host that might need to be removed: {...}.
  WARNING nova.compute.resource_tracker [...] Instance <server-id> is not being actively managed by this compute host but has allocations referencing this compute host: {...}. Skipping heal of allocation because we do not know what to do.
  ```

- Deleting the resource provider of a decommissioned compute fails:

  ```text
  $ openstack resource provider delete <rp-uuid>
  Unable to delete resource provider <rp-uuid>: Resource provider has allocations. (HTTP 409)
  ```

## Impact

- Leftover allocations: capacity is lost. The scheduler skips computes that are in fact free; builds fail with
  `No valid host`, or concentrate on fewer computes.
- Missing allocations: the scheduler believes a compute has room it does not have. It overcommits it beyond the
  allocation ratios; instances on it compete for CPU and memory, and a build can fail on the compute itself.
- Running instances are not stopped by either. Quota usage is not affected: Nova counts quota from its own
  database by default (`[quota] count_usage_from_placement = False`).

## Quick checks

Read-only. Run with admin credentials, from any host with the `openstack` client and the `osc-placement` plugin.

```bash
# 1. The compute's resource provider, its inventory and its usage
openstack resource provider list --name <hypervisor-hostname>
openstack resource provider inventory list <rp-uuid>
openstack resource provider usage show <rp-uuid>

# 2. Who holds the allocations on that provider (each key is a consumer: an instance or a migration UUID)
openstack resource provider show --allocations <rp-uuid>

# 3. What Nova thinks runs there
openstack server list --all-projects --host <host> -c ID -c Name -c Status -c Flavor

# 4. Where one instance's allocations are (expected: one provider, the instance's compute)
openstack --os-placement-api-version 1.12 resource provider allocation show <server-id>
```

Then let Nova compare the two sides. `audit` without `--delete` changes nothing; `heal_allocations --dry-run`
changes nothing and should exit with return code 4.

**Kolla-Ansible:** on a controller,
`sudo docker exec nova_api nova-manage placement audit --verbose` and
`sudo docker exec nova_api nova-manage placement heal_allocations --dry-run --verbose`.
The `nova_api` container has the `[api_database]` and `[placement]` sections both commands need.

**Any:** `nova-manage placement audit --verbose` and `nova-manage placement heal_allocations --dry-run --verbose`
on a host whose `nova.conf` has `[api_database] connection` and `[placement]` set.

Expected output when something is out of sync (strings from `nova/cmd/manage.py`):

```text
$ nova-manage placement audit --verbose
Allocations were set against consumer UUID <consumer-uuid> but no existing instances or active migrations are related.
Allocations for consumer UUID <consumer-uuid> on Resource Provider <rp-uuid> can be deleted: {'VCPU': 2, 'MEMORY_MB': 4096, 'DISK_GB': 20}
Processed 1 allocation.
$ echo $?
3

$ nova-manage placement heal_allocations --dry-run --verbose
Running batches of 50 until complete
Looking for instances in cell: <cell-uuid>
Found 4 candidate instances.
The allocation of instance <server-id> is up-to-date. Nothing to be healed.
[dry-run] Create allocations for instance <server-id>: {"allocations": {"<rp-uuid>": {"resources": {"VCPU": 1, "MEMORY_MB": 512, "DISK_GB": 1}}}, "project_id": "<project-id>", "user_id": "<user-id>", "consumer_generation": null}
Processed 0 instances.
```

## Diagnosis

### Which way the drift goes

Compare check 2 and check 3:

| What you find | What it is |
|---|---|
| A consumer on the provider that is not an instance at all (`openstack server show <consumer-uuid>` finds nothing), and `audit` lists it | Orphaned allocation: the instance or the migration no longer exists |
| An instance listed on another host, whose allocation is still on this provider too (check 4 shows two providers, no move in progress); `audit` lists it on this provider as well | Leftover source allocation after an evacuation or a failed move |
| An instance on this host with no allocation at all (check 4 empty), and `heal_allocations --dry-run` proposes to create it | Missing allocation |
| An instance with `task_state` set (migrating, resizing, rebuilding for an evacuation) holding allocations on two providers | Normal while the move runs: the migration's UUID holds the source allocation, the instance's UUID the destination one. Until the instance's host switches to the destination, `audit` lists the destination one as orphaned: do not touch it |

### What `audit` and `heal_allocations` look at

- `nova-manage placement audit` walks every resource provider (or one, with `--resource_provider <rp-uuid>`). For
  each consumer whose allocation holds a Nova resource class (`VCPU`, `PCPU`, `MEMORY_MB`, `DISK_GB`, `VGPU`,
  bandwidth classes), it checks whether the consumer is one of the instances whose host and node are that compute
  node, or one of the in-progress migrations from or to that node (`_get_instances_and_current_migrations`). If
  neither, it reports the allocation as orphaned. So it reports, on a given provider:
  - allocations of instances that no longer exist (cause 1);
  - the leftover allocation of an instance that now runs on another compute (cause 2);
  - during a cold migration, a resize or an evacuation, the allocation that the *instance* UUID holds on the
    destination while the instance's host is still the source. This one is not orphaned.

  Return codes: 0 nothing found, 3 orphans found, 4 orphans found and deleted, 1 error, 127 invalid input.
- `nova-manage placement heal_allocations` walks the instances of every cell except cell0. It creates the
  allocation of an instance that has none, from its flavor, on its compute node, and fills in a missing project or
  user on an existing allocation and port resource requests (unless `--skip-port-allocations`). It skips
  instances with a `task_state` (`Instance <server-id> is undergoing a task state transition: <task_state>`) and
  instances not on a host. It does not move or remove an allocation that sits on the wrong provider. Return codes:
  0 allocations created, 4 nothing to do, 1 `--max-count` reached with more to process, 2 compute node not found,
  3 Placement create or update failed, 5 to 7 Neutron port errors, 8 instance with vGPU or Cyborg resources (not
  supported), 127 invalid input. Like every `nova-manage` command, both return 255 on an unexpected error, with the
  traceback after `An error has occurred:`.

### Why Nova did not clean it up itself

The resource tracker of each `nova-compute` deletes, at each resource update, the allocations against its own node
of instances that are marked deleted. It leaves the others alone and only logs them (the lines in *Symptom*):

- a consumer not found in the database at all (an instance archived, or an allocation created by hand) is logged
  at `INFO` and never deleted, to avoid racing with a build in progress;
- an instance moved to another host is logged at `WARNING` and not deleted.

After an evacuation, the source compute removes the source allocations of evacuated instances when its
`nova-compute` starts again (`Cleaning up allocations of the instance as it has been evacuated from this host`).
If the source compute never comes back, nobody removes them.

## Possible causes

### Cause 1: allocations of instances that no longer exist

The allocation outlived its instance. A delete while the compute is down (a local delete) normally removes the
allocations itself (`_local_delete` in `nova/compute/api.py`), so it is a cause only when that call failed, for
example with Placement unreachable at that moment (`Allocation delete failed during local delete cleanup.` in the
`nova-api` log). Other ways: the instance was archived with `nova-manage db archive_deleted_rows` before its
allocation was removed, or the allocation was created by another tool. `audit` reports it; the compute log repeats
`has allocations against this compute host but is not found in the database`.

### Cause 2: source allocations left after an evacuation or a failed move

The instance runs on another compute, and check 4 shows allocations on both the old and the new provider, with no
move in progress. Typical after an evacuation from a compute that never restarted, or after a resize or migration
that failed and was reset by hand. The old compute logs `has been moved to another host` if its `nova-compute` runs.
`audit` reports it on the old provider, because the instance is not on that node.

### Cause 3: instances without allocations

The Placement database was lost or restored from an older backup, or an allocation was deleted by hand. The usage of
the compute is lower than what runs on it. `heal_allocations --dry-run` lists the instances it would heal.

## Resolution

Before any change, save what Placement holds now, so you can set it back by hand. For each consumer you are going
to change (each consumer `audit` lists, each instance you heal), save its whole allocation, with every provider,
the project and the user, which `allocation set` needs:

```bash
openstack resource provider show --allocations <rp-uuid> -f json > allocations-<rp-uuid>.json
openstack --os-placement-api-version 1.12 resource provider allocation show <consumer-uuid> -f json > allocation-<consumer-uuid>.json
```

Then check that no move is running anywhere, since `audit` takes the destination allocation of an instance being
moved for an orphan:

```bash
# every migration in a final state: confirmed, reverted, error, failed, completed, cancelled or done
openstack server migration list
# no instance with a task state
openstack server list --all-projects --long -c ID -c "Task State" -f value | awk '$2 != "None"'
```

### For cause 1: delete the orphaned allocations

Run `audit` without `--delete` first, and sort each consumer it lists:

- `openstack server show <consumer-uuid>` finds no server, and `openstack server migration list` does not show it:
  cause 1, delete;
- it is an instance running on another compute, with no task state: cause 2, the leftover on this provider can be
  deleted the same way (below) or with `allocation unset`;
- it is an instance or a migration in a move: leave it, and do not run `--delete` on this provider until the move
  has ended.

> 🛑 **Destructive.** For each consumer `audit` reports on the audited provider, removes its allocation on that
> provider: only that provider's part when the consumer also has allocations elsewhere, the whole allocation when
> this provider is its only one. Without `--resource_provider` it does this on every provider. What can be lost: the
> destination allocation of an instance being moved (the destination is then overcommitted, and the instance has no
> allocation once the move ends), and a migration's allocation that `audit` failed to match; `heal_allocations`
> restores neither. Who is affected: the scheduler sees the freed capacity at once. Precondition: every migration in
> a final state and no instance with a task state (the checks above), and the saved JSON of each listed consumer.
> Undo: recreate the allocation by hand with `openstack resource provider allocation set` from the saved JSON
> *(not run on the lab)*.

**Kolla-Ansible:**

```bash
sudo docker exec nova_api nova-manage placement audit --verbose --delete --resource_provider <rp-uuid>
```

**Any:** `nova-manage placement audit --verbose --delete --resource_provider <rp-uuid>`.

Expected: `Deleted allocations for consumer UUID <consumer-uuid> on Resource Provider <rp-uuid>: {...}` and return
code 4. Always pass `--resource_provider`: it keeps the change to the provider you checked.

### For cause 2: remove only the stale provider from the instance's allocation

The instance keeps its allocation on the compute where it runs; only the part on the old provider goes.

> ⚠️ **State-changing.** Removes the instance's allocation on `<old-rp-uuid>` only. Precondition: the instance runs
> on the other compute and no move is in progress for it (`task_state` empty). Undo: set the allocation back from
> the saved JSON with `openstack resource provider allocation set` *(not run on the lab)*.

```bash
openstack --os-placement-api-version 1.12 resource provider allocation unset --provider <old-rp-uuid> <server-id>
```

`allocation unset` needs osc-placement 1.8.0 or later. `nova-manage placement audit --delete --resource_provider
<old-rp-uuid>` does the same for every leftover on that provider (it removes only that provider's part of a
consumer that also has allocations elsewhere), with the same 🛑 preconditions as for cause 1. Do not use `openstack resource provider allocation delete`
here: it removes the instance's allocations on every provider, including the one it runs on.

If the old compute is decommissioned for good, its provider can be deleted once it holds no allocation:
`openstack resource provider delete <old-rp-uuid>`.

### For cause 3: create the missing allocations

> ⚠️ **State-changing.** Creates allocations from each instance's flavor on its compute node. The scheduler sees
> less free capacity at once; a compute already overcommitted beyond its ratios will look full. Precondition: run
> with `--dry-run` first and read what it proposes. Undo: `openstack resource provider allocation delete <server-id>`
> for an instance whose allocation was wrong, then heal it again.

**Kolla-Ansible:**

```bash
sudo docker exec nova_api nova-manage placement heal_allocations --verbose --instance <server-id>
# or every instance, in batches of 50 (not run on the lab):
sudo docker exec nova_api nova-manage placement heal_allocations --verbose
```

**Any:** the same `nova-manage placement heal_allocations` commands.

Expected: `Successfully created allocations for instance <server-id>.` then `Processed <n> instances.` and return
code 0; return code 4 if there was nothing to heal. Instances with vGPU or Cyborg device profiles are refused (return
code 8): their allocations are nested and must be set by hand.

### When one instance's allocations are wrong in a way neither tool fixes

First try `heal_allocations --force`: for one instance with no task state, it replaces its allocations with the
ones computed from its flavor on its compute node, without deleting them first.

> ⚠️ **State-changing.** Replaces the instance's allocations with flavor-based ones on its compute node.
> Precondition: no task state on the instance, and its allocation saved as JSON above. Undo: `openstack resource
> provider allocation set` from the saved JSON *(not run on the lab)*.

```bash
sudo docker exec nova_api nova-manage placement heal_allocations --verbose --instance <server-id> --force
```

If that is not enough, the upstream procedure is to delete all its allocations and heal it.

> 🛑 **Destructive.** Deletes every allocation of the consumer, on every provider. What can be lost: a migration
> allocation if the instance is in a move (the move then fails or overcommits its destination), nested allocations
> (vGPU, Cyborg, child providers from `provider.yaml`) that `heal_allocations` cannot recreate. Precondition: the
> instance is `ACTIVE`, `SHUTOFF` or `PAUSED` with no `task_state`, and its allocation is saved as JSON above.
> Undo: `openstack resource provider allocation set` from the saved JSON *(not run on the lab)*.

```bash
openstack resource provider allocation delete <server-id>
sudo docker exec nova_api nova-manage placement heal_allocations --verbose --instance <server-id>
```

## Verification

```bash
# No orphan left (return code 0, no output)
sudo docker exec nova_api nova-manage placement audit --verbose; echo $?

# Nothing left to heal: no "[dry-run] Create" or "[dry-run] Update" line
#   (the return code is 4 with --dry-run whatever it finds, so it proves nothing)
sudo docker exec nova_api nova-manage placement heal_allocations --dry-run --verbose | grep -e "\[dry-run\] Create" -e "\[dry-run\] Update"
# expected: no output

# The compute's usage matches the flavors of its instances
openstack resource provider usage show <rp-uuid>

# Each repaired instance has one allocation, on the provider of its compute
openstack --os-placement-api-version 1.12 resource provider allocation show <server-id>
```

At the next resource update (by default every 60 seconds, `[DEFAULT] update_resources_interval = 0`), the `nova-compute` log of the compute no longer prints the
lines quoted in *Symptom* for these consumers. If builds failed with `No valid host`, create a test instance and
check that it is scheduled.

## Lab observations

On a Kolla-Ansible 22.2.0 lab (2026.1, 3 controllers, 2 computes), the three causes were injected by hand with
osc-placement 4.9.1: an allocation for a made-up consumer on compute-1 (cause 1), a second provider (compute-2) added
to the allocation of an instance running on compute-1 (cause 2), and the allocation of an instance on compute-2
deleted (cause 3).

- Within 20 s, the resource trackers logged, then repeated at each update until the repair, the `INFO ... is not
  found in the database` line on compute-1 and the `WARNING ... has been moved to another host` line on compute-2,
  as quoted in *Symptom*. Nothing was logged for the instance without allocation.
- `audit --verbose` listed causes 1 and 2 (return code 3); `heal_allocations --dry-run --verbose` proposed cause 3
  (return code 4, `Processed 0 instances.`). The cell was printed as its UUID only.
- `audit --verbose --delete --resource_provider <rp-uuid>` removed the cause 1 allocation (return code 4);
  `allocation unset --provider` left only the compute-1 part; `heal_allocations --verbose --instance` created the
  missing one (return code 0). `--force` (`Force flag passed for instance <server-id>`, `Successfully updated
  allocations`) and `allocation delete` followed by a heal also worked, return code 0.
- Verification: `audit` returned 0 with no output, the dry-run grep printed nothing, each instance had one allocation
  on its compute; the log lines stopped.

## Validity and limits

- Written from Nova 2026.1: `nova/cmd/manage.py` (`PlacementCommands.audit` and `heal_allocations`, their messages
  and return codes), `nova/compute/resource_tracker.py` (`_remove_deleted_instances_allocations`),
  `nova/compute/manager.py` (`_destroy_evacuated_instances`), `nova/scheduler/manager.py`, and the upstream
  *Orphaned resource allocations* guide. Kolla-Ansible 2026.1 `nova.conf.j2` for the `nova_api` container. Run on a
  lab with the causes injected by hand in Placement (see *Lab observations*), not produced by a real evacuation,
  failed move or database restore. Not run: the undo with `allocation set` from the saved JSON, `heal_allocations`
  over every instance, the `No valid host` symptom, the `resource provider delete` refusal.
- The `nova-compute` log lines are shown with the logger and message from the source; the timestamp, process and
  request id prefix depends on your logging configuration.
- `audit` matches consumers against instances and active migrations only. A consumer created by another service
  with a Nova resource class is reported as orphaned: check it before `--delete`.
- Not covered: Ironic nodes, nested providers (vGPU, PCI in Placement, Cyborg), shared storage providers, and
  `count_usage_from_placement = True`, where quota usage also follows these allocations.

## Related

- [No valid host was found, although capacity exists](no-valid-host-with-free-capacity.md): where inflated usage
  shows up first.
- [A compute was renamed, reinstalled or re-added](compute-renamed-or-re-registered-resource-provider-conflict.md):
  when the provider itself is the leftover.
- [Stale records](../lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md): the wider clean-up of
  instances, ports and attachments, of which phantom allocations are one part.
- [Compute host unreachable, fence before you evacuate](../recovery/compute-host-failure-fence-before-evacuate.md):
  the evacuation that leaves source allocations behind.
- Nova 2026.1: [Orphaned resource allocations](https://docs.openstack.org/nova/2026.1/admin/troubleshooting/orphaned-allocations.html),
  [nova-manage placement](https://docs.openstack.org/nova/2026.1/cli/nova-manage.html#placement-commands).

---

*Provenance: written by agent:ozarland-author, from the Nova and Kolla-Ansible 2026.1 documentation and source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=scheduling/placement-allocations-out-of-sync-audit-and-heal.md&title=Feedback%3A+scheduling/placement-allocations-out-of-sync-audit-and-heal.md):
your release, your deployment method, and what happened.
