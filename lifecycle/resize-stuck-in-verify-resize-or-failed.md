---
title: Resize or cold migration not finished. Instance in VERIFY_RESIZE, stuck in RESIZE, or in ERROR after a failed resize
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [nova, placement, libvirt]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any multinode
  network: any
  storage: local instance disks or shared storage
  os: any
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/user/resize.html
  - https://docs.openstack.org/nova/2026.1/admin/configuration/resize.html
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#DEFAULT.resize_confirm_window
  - https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/server.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Resize or cold migration not finished. Instance in VERIFY_RESIZE, stuck in RESIZE, or in ERROR after a failed resize

## Symptom

After `openstack server resize --flavor <flavor> <server-id>`, or a cold migration (`openstack server migrate`), the
instance is in one of three states:

```text
$ openstack server show <server-id> -c status -c OS-EXT-STS:task_state -c OS-EXT-SRV-ATTR:host -c flavor
```

| `status` | `task_state` | What it means |
|---|---|---|
| `VERIFY_RESIZE` | `None` | The resize **worked** and waits for a confirm or a revert. It can stay so for days |
| `RESIZE` | `resize_prep`, `resize_migrating`, `resize_migrated` or `resize_finish` | The operation stopped part way: a service died, or a message was lost |
| `ERROR` | `None` | A step failed; the fault says which one |

## Impact

One instance. In `VERIFY_RESIZE`, it runs on the new flavor (and host), but the old copy is kept: its disks stay on
the source compute, and Placement keeps resources allocated on both computes until the confirm. Many instances left
so can make the scheduler see computes as fuller than they are. In `RESIZE` or `ERROR`, the instance may be shut off,
and its owner cannot act on it.

## Quick checks

Read-only. With admin credentials:

```bash
# 1. The state, host, flavor, and fault if any
openstack server show <server-id> -c status -c OS-EXT-STS:task_state -c OS-EXT-STS:power_state \
  -c OS-EXT-SRV-ATTR:host -c flavor -c fault

# 2. The migration record: source and destination computes, status (finished, error, migrating...)
openstack server migration list --server <server-id>

# 3. The instance's actions, and the failed step
openstack server event list <server-id>
openstack server event show <server-id> <request-id>

# 4. Are both computes up?
openstack compute service list --service nova-compute
```

The `nova-compute` logs of the source and destination computes, for the server ID, have the error:

**Kolla-Ansible:** `sudo grep <server-id> /var/log/kolla/nova/nova-compute.log`

**Any:** the `nova-compute` service log on each compute.

## Diagnosis

### How a resize runs (Nova 2026.1, `nova/compute/manager.py`)

1. **Destination, `prep_resize`:** claims resources on the destination. `task_state` `resize_prep`.
2. **Source, `resize_instance`:** `task_state` `resize_migrating`. The libvirt driver powers the guest off, unplugs
   its ports, disconnects its volumes, renames the instance directory to `<uuid>_resize`, and copies the local disks
   to the destination over SSH (or uses the shared instance directory). Then the instance's `host` is set to the
   **destination**, and `task_state` becomes `resize_migrated`.
3. **Destination, `finish_resize`:** switches the instance to the new flavor, resizes and starts the guest, and sets
   the instance to `RESIZED` (`VERIFY_RESIZE` in the API), migration status `finished`.
4. **Confirm** (`openstack server resize confirm`): deletes the old copy on the source and the source allocation.
   **Revert** (`openstack server resize revert`): deletes the new guest, restarts the old one on the source with the
   old flavor. Both are allowed only in `VERIFY_RESIZE`.

`[DEFAULT] resize_confirm_window` (default 0, disabled) makes the **destination** compute confirm resizes older than
that many seconds. Kolla-Ansible 2026.1 does not set it, and sets `allow_resize_to_same_host = true`.

### What fails where

| Step | Usual errors (fault or compute log) | State after |
|---|---|---|
| `prep_resize` | No valid host, resources claim failed, anti-affinity | `ACTIVE` again (or `SHUTOFF`), migration `error` |
| Disk copy | `Unable to resize disk down.`, `not able to execute ssh command`, `Migration is not supported for LVM backed instances` | Rolled back to the previous state: these errors are raised as `InstanceFaultRollback` |
| Disk copy, other errors | copy interrupted, disk full on the destination | `ERROR`, `host` still the **source** |
| `finish_resize` | the guest fails to start on the destination (libvirt, volume, network) | `ERROR`, `host` the **destination**, flavor set back to the old one |

A `nova-compute` that restarts while an instance is in `resize_migrating` reverts the instance on that host and
clears the task state (`_init_instance`). A restart in `resize_prep` clears the task state only.

### What the compute logs say

Messages of `nova/compute/manager.py` (2026.1), in the `nova-compute` log of the compute named:

| Message | Where | Meaning |
|---|---|---|
| `Migrating` | source | The disk copy starts (`_resize_instance`) |
| `Setting instance back to <state> after: <error>` | source or destination | A rollback error (`InstanceFaultRollback`): the instance is back in its previous state, `<error>` says why |
| `Setting instance vm_state to ERROR` followed by a traceback | source or destination | Any other error: the instance is in `ERROR`; the traceback is the cause |
| `Instance found in migrating state during startup. Resetting task_state` | source | `nova-compute` restarted during the copy and reverted the instance (`_init_instance`) |
| `Failed to revert crashed migration` | source | That revert failed: recover by hand (below) |
| `Found <n> unconfirmed migrations older than <s> seconds`, then `Automatically confirming migration <id> for instance <uuid>` | destination | `resize_confirm_window` at work |
| `Setting migration <id> to error: In states <vm_state>/<task_state>, not RESIZED/None` | destination | The automatic confirm skipped an instance that was not in `VERIFY_RESIZE` |
| `Confirming resize on source host.` / `Reverting resize on destination host.` | source / destination | A confirm or revert running |

## Possible causes

### Cause 1: nobody confirmed (VERIFY_RESIZE)

Nothing failed. The owner, or the tool that ran the resize, never confirmed it.

### Cause 2: a service died or a message was lost (RESIZE with a task state)

`nova-compute` on the source or destination stopped, the message bus failed, or the conductor did not get a reply.
The migration record stays in `pre-migrating`, `migrating` or `post-migrating`.

### Cause 3: SSH between computes (ERROR or rollback at the disk copy)

With local instance disks, the source copies the disks to the destination over SSH as the `nova` user. With
Kolla-Ansible, through the `nova_ssh` containers, which must run on every compute and accept the keys of the others.

### Cause 4: the destination cannot run the guest (ERROR after finish)

A problem on the destination compute: libvirt or QEMU error, a volume connection failure, a port binding failure,
no space for the disks.

### Cause 5: a smaller disk (rolled back)

`Unable to resize disk down.`: the new flavor's root or ephemeral disk is smaller than the old one. Nova does not
shrink disks, except a root disk booted from volume.

## Resolution

### For cause 1: confirm or revert

Ask the owner first: the confirm deletes the old copy for good.

> ⚠️ **State-changing.** `confirm` deletes the instance's previous copy on the source compute and frees its
> resources; the revert is no longer possible. `revert` destroys the resized guest and restarts the previous one on
> the source, with the previous flavor: changes made inside the guest since the resize are lost if they were on its
> local disk.

```bash
openstack server resize confirm <server-id>      # or: openstack server migration confirm <server-id>
openstack server resize revert <server-id>       # or: openstack server migration revert <server-id>
```

To confirm automatically after a delay, set `resize_confirm_window` on the computes:

**Kolla-Ansible:** in `/etc/kolla/config/nova/nova-compute.conf`, `[DEFAULT] resize_confirm_window = <seconds>`,
then `kolla-ansible reconfigure -i <inventory> --tags nova`.

> ⚠️ **State-changing.** `reconfigure` restarts the `nova_compute` containers; running instances are not affected.
> From then on, every resize older than the window is confirmed without the owner, and can no longer be reverted.

**Any:** the same option in `nova.conf` on every compute, then restart `nova-compute`.

### For cause 2: restart the service, then check again

Bring back the service that stopped (quick check 4), or the message bus
([RabbitMQ](https://github.com/OzarLand/openstack-vm-runbooks/blob/main/recovery/rabbitmq-partition-or-alarm-rpc-timeouts.md),
[shared dependencies](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md)). If the instance
was in `resize_migrating` on that compute, its restart reverts the instance. Then check quick check 1 again.

If the instance stays in `RESIZE` with the services up, find where its disks are (the `host` field and the migration
record), and on that compute, whether the guest exists:

**Kolla-Ansible:** `sudo docker exec nova_libvirt virsh list --all --uuid | grep <server-id>`

**Any:** `sudo virsh list --all --uuid | grep <server-id>`

Then reset the state and hard reboot, as for an instance in `ERROR` (below).

### For causes 3 and 4: fix the compute, then recover the instance

Fix what the error names: the `nova_ssh` containers and keys (cause 3), or the destination's own problem (cause 4,
with the runbooks of [boot from volume](../scheduling/instance-fails-to-boot-from-volume.md),
[port binding](instance-stuck-in-build-waiting-for-vif-plugged.md), [disk full](compute-disk-full-instance-paused.md)).

Then recover the instance where Nova says it is (`host`): the python-openstackclient help for `server set --state`
says that a hard reboot is the right first step for an instance in `ERROR`, and that resetting the state after a
failed move operation can leave an instance unusable.

> ⚠️ **State-changing.** Recreates the guest on the compute in the instance's `host` field, from the disks there,
> with the flavor Nova records. Precondition: the instance's disks are on that compute (local disks), or on shared
> storage or volumes. If the disks are only on the other compute, stop here: see *Validity and limits*.

```bash
openstack server reboot --hard <server-id>
```

If the instance is still in a `RESIZE` task state, the reboot is refused; clear the state first:

> ⚠️ **State-changing.** Sets the instance to `ACTIVE` with no task state in the database only; nothing changes on
> the compute. Undo: `openstack server set --state error <server-id>`.

```bash
openstack server set --state active <server-id>
openstack server reboot --hard <server-id>
```

The source compute keeps the previous copy as `<instances-path>/<server-id>_resize` after a failed resize. Leave it
until the instance runs and its owner has checked it; it is the only copy if the recovery goes wrong.

### For cause 5: choose a flavor with a disk at least as large

Nothing to recover: the instance is back in its previous state. Resize again to a flavor whose root and ephemeral
disks are at least the current sizes.

## Verification

```bash
openstack server show <server-id> -c status -c OS-EXT-STS:task_state -c OS-EXT-SRV-ATTR:host -c flavor
# expected: ACTIVE (or SHUTOFF if it was), no task state, the expected flavor and host
openstack server migration list --server <server-id>
# expected: the last migration confirmed, reverted or error, none in progress
```

Then check the allocations: after a failed or abandoned resize, the migration may still hold resources on the
source compute ([no valid host with free capacity](../scheduling/no-valid-host-with-free-capacity.md) and
[stale records](stale-records-stuck-instances-orphan-ports-attachments.md) show how to find them).

## Validity and limits

- Written from Nova 2026.1 (`nova/compute/manager.py`, `nova/compute/api.py`, `nova/virt/libvirt/driver.py`, the
  resize guides, `resize_confirm_window`), python-openstackclient 2026.1 (`server resize`, `server migration`,
  `server set --state`) and Kolla-Ansible 2026.1 (`nova.conf.j2`, `nova_ssh`). Not reproduced on a lab.
- With local disks, an instance in `ERROR` after a failed copy whose `host` points to a compute without its disks
  is not covered: moving the disks by hand is outside this page.
- Not covered: cross-cell resize, resize of instances with PCI devices, vGPUs or NUMA topology, and VMware or
  Ironic drivers.

## Related

- [Moving instances off a compute host for planned maintenance](../migration/moving-instances-off-a-host-for-maintenance.md):
  cold migration as a maintenance tool, and its confirm.
- [Stale records](stale-records-stuck-instances-orphan-ports-attachments.md): resetting states and orphan
  allocations.
- [New instance in ERROR with "Exceeded maximum number of retries"](../scheduling/build-fails-exceeded-maximum-number-of-retries.md)
- Nova 2026.1: [resize an instance](https://docs.openstack.org/nova/2026.1/user/resize.html),
  [resize configuration](https://docs.openstack.org/nova/2026.1/admin/configuration/resize.html)

---

*Provenance: written by agent:ozarland-author, from the Nova, python-openstackclient and Kolla-Ansible 2026.1
documentation and source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=lifecycle/resize-stuck-in-verify-resize-or-failed.md&title=Feedback%3A+lifecycle/resize-stuck-in-verify-resize-or-failed.md):
your release, your deployment method, and what happened.
