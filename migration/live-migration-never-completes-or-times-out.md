---
title: Live migration runs for a long time, then is aborted. The instance writes memory faster than it can be copied
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [nova, libvirt, qemu]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any multinode
  network: any
  storage: any
  os: any
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/admin/live-migration-usage.html
  - https://docs.openstack.org/nova/2026.1/admin/configuring-migrations.html
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#libvirt.live_migration_completion_timeout
  - https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/server-migration.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Live migration runs for a long time, then is aborted. The instance writes memory faster than it can be copied

## Symptom

A live migration (`openstack server migrate --live-migration <server-id>`) was accepted and started, but does not
finish. The instance stays in `MIGRATING`, minutes or hours longer than its peers took:

```text
$ openstack server show <server-id> -c status -c OS-EXT-SRV-ATTR:host
| status                 | MIGRATING     |
| OS-EXT-SRV-ATTR:host   | <source-host> |
```

Then, eventually, the instance is `ACTIVE` again **on the source host**, and the migration record says it failed:

```text
$ openstack server migration list --server <server-id>
| … | <source-host> | <dest-host> | … | error | … | live-migration | …
```

A migration refused at once (a pre-check error, shared storage, CPU, versions) is a different case: see the
[maintenance runbook](moving-instances-off-a-host-for-maintenance.md).

## Impact

One instance at a time. It keeps running on the source the whole time, so its users see nothing until the end, but:

- the source host cannot be drained, which blocks the maintenance it was for;
- the migration loads the network and, for a block migration, the disks of both hosts while it runs;
- the fixes below slow the instance down (auto-converge), pause it (force-complete) or tie it to both hosts for a
  while (post-copy).

## Quick checks

Read-only. With admin credentials:

```bash
# 1. The migration record and its id (status running, then error or cancelled)
openstack server migration list --server <server-id>

# 2. Its progress (needs compute API 2.24 or later). Repeat a few times: is memory_remaining_bytes going down?
openstack --os-compute-api-version 2.24 server migration show <server-id> <migration-id>

# 3. The instance's size: memory, and ephemeral disk for a block migration
openstack server show <server-id> -c flavor
```

On the **source** compute, the `nova-compute` log says when Nova gave up:

**Kolla-Ansible:**

```bash
sudo grep -E 'WARNING.*<server-id>' /var/log/kolla/nova/nova-compute.log | grep -i 'live migration not completed'
```

**Any:**

```bash
sudo grep -E 'WARNING.*<server-id>' <nova-compute-log> | grep -i 'live migration not completed'
```

The upstream guide shows the line as `live migration not completed after <n> sec`.

## Diagnosis

### Nova and libvirt

A live migration copies the instance's memory to the destination while the instance keeps running. A page written
again after it was copied must be copied again. If the instance dirties memory faster than the network copies it,
`memory_remaining_bytes` stops going down and the migration cannot converge (Nova's live migration usage guide).

Nova then applies its completion timeout: `[libvirt] live_migration_completion_timeout` (default 800) seconds **per
GiB** of instance memory, plus the ephemeral disk size for a block migration. An 8 GiB instance on shared storage
times out after 6400 s with the defaults. What happens then is `[libvirt] live_migration_timeout_action`:

- `abort` (default): the migration is cancelled; the instance stays on the source;
- `force_complete`: Nova pauses the instance to finish the copy, or switches to post-copy if it is allowed.

Kolla-Ansible 2026.1 does not set these options, nor `live_migration_permit_auto_converge` or
`live_migration_permit_post_copy`: Nova's defaults apply (auto-converge and post-copy off).

### Signatures

| You see | Meaning |
|---|---|
| `memory_remaining_bytes` falls steadily, slowly | The copy progresses but the link is slow: a large instance on a slow or busy migration network |
| `memory_remaining_bytes` falls, then hovers at the same order of magnitude | The instance dirties memory as fast as it is copied: it will not converge without help |
| `disk_*` fields stay at 0 during a block migration | Expected: the guide notes that disk bytes are not reported |
| `live migration not completed after <n> sec` in the source log | Nova's completion timeout fired |

## Possible causes

### Cause 1: memory written faster than it is copied

A busy database, cache or in-memory workload. The migration hovers and is aborted at the timeout.

### Cause 2: a slow or shared migration network

The copy rate is low: the migration network is 1 Gbit/s, shared with other traffic, or several migrations run at
once. With Kolla-Ansible, migrations use `migration_interface`, which defaults to `api_interface`
(`group_vars/all/nova.yml`), so they share the API and database network unless it is set otherwise.

### Cause 3: a block migration of a large local disk

The disk is copied as well as the memory, and the timeout includes the disk size. It takes long by nature.

## Resolution

Start with the least intrusive step. Every step below changes something; none deletes anything.

### Stop a migration that will not converge

> ⚠️ **State-changing.** Cancels the running migration. The instance keeps running on the source, unchanged; the
> load on the network and disks stops. Nothing to undo: start a new migration later.

```bash
openstack --os-compute-api-version 2.24 server migration abort <server-id> <migration-id>
```

### Finish this one migration now (cause 1)

> ⚠️ **State-changing.** Force-complete pauses the instance until the remaining memory is copied (or switches to
> post-copy if it is enabled). The pause affects the instance's timekeeping, and the upstream guide warns that not
> every application tolerates it. Agree it with the instance's owner first.

```bash
openstack --os-compute-api-version 2.22 server migration force complete <server-id> <migration-id>
```

### Let migrations converge in future (causes 1 and 2)

Both are libvirt features that Nova only uses when allowed, on **all** computes:

- **auto-converge:** libvirt throttles the instance's CPU until the copy outpaces its writes. The instance slows
  down, and the guide says it does not guarantee success;
- **post-copy:** the instance moves to the destination before all its memory is copied, and fetches missing pages
  from the source. The guide says it is certain to complete, but if the network between the hosts breaks during
  that phase, the instance is rebooted.

> ⚠️ **State-changing.** `reconfigure` restarts the `nova_compute` containers whose configuration changed; running
> instances are not affected. Undo: remove the override and reconfigure again.

**Kolla-Ansible:**

```ini
# /etc/kolla/config/nova/nova-compute.conf
[libvirt]
live_migration_permit_auto_converge = true
# or, accepting the risk above:
# live_migration_permit_post_copy = true
```

```bash
kolla-ansible reconfigure -i <inventory> --tags nova
```

**Any:** set the same options in `nova.conf` on every compute and restart `nova-compute`.

For cause 2, give migrations their own network: `migration_interface` in `globals.yml` (or per host), then a
`reconfigure`, in a maintenance window. For cause 3, plan the time, or use cold migration for that instance.

## Verification

```bash
openstack server migration list --server <server-id>
# expected: the new migration completed
openstack server show <server-id> -c status -c OS-EXT-SRV-ATTR:host
# expected: ACTIVE on the destination host
```

Then check the instance's application with its owner, especially after a force-complete pause.

## Lab observations

None for this failure. On the lab of the [maintenance runbook](moving-instances-off-a-host-for-maintenance.md)
(Kolla-Ansible 22.2.0, two computes, small test instances), live block migrations completed in 32 to 34 s: the
instances were too small and idle to show this problem.

## Validity and limits

- Written from Nova 2026.1 (live migration usage and configuring migrations guides, the `[libvirt]` options),
  python-openstackclient 2026.1 (`server migration` commands and their microversions) and Kolla-Ansible 2026.1 (no
  override of these options; `migration_interface`). Not reproduced on a lab.
- The timeout figures are Nova's defaults; a deployment may have changed them.
- Not covered: migrations refused by a pre-check (see the maintenance runbook), failures after the switch-over, live
  migration with PCI devices, vGPU or NUMA-pinned instances, and QEMU-native TLS for migration streams.

## Related

- [Moving instances off a compute host for planned maintenance](moving-instances-off-a-host-for-maintenance.md):
  choosing live, block or cold migration, and the pre-check failures.
- [CPU steal on pinned instances](cpu-steal-on-pinned-instances.md)
- Nova 2026.1: [live-migrate instances](https://docs.openstack.org/nova/2026.1/admin/live-migration-usage.html),
  [configuring migrations](https://docs.openstack.org/nova/2026.1/admin/configuring-migrations.html)

---

*Provenance: written by agent:ozarland-author, from the Nova, python-openstackclient and Kolla-Ansible 2026.1
documentation and source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=migration/live-migration-never-completes-or-times-out.md&title=Feedback%3A+migration/live-migration-never-completes-or-times-out.md):
your release, your deployment method, and what happened.
