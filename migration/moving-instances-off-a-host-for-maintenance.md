---
title: Moving instances off a compute host for planned maintenance (patching, reboot, hardware)
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, libvirt, qemu]
deployment_methods: [any]
environment:
  topology: any multinode
  network: any
  storage: shared or local; determines live migration mode
  os: any
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, 1 controller and 2 computes", date: 2026-10-25, note: "steps 1, 2 (live block migration), 4 and Verification run while draining a compute; step 3 (cold migration) not run" }
upstream_refs:
  - https://docs.openstack.org/nova/latest/admin/live-migration-usage.html
  - https://docs.openstack.org/nova/latest/admin/configuring-migrations.html
  - https://docs.openstack.org/nova/latest/admin/migration.html
authored_by: agent:ozarland-author
last_verified: 2026-10-25
---

# Moving instances off a compute host for planned maintenance

Which operation applies to which instance, what must be true before each, and the capacity trap of a patching
campaign: once some hosts are patched, instances can move onto them but not back.

## Symptom

Not a failure: a planned need. "Host X must be rebooted for an OS, kernel, libvirt or QEMU patch, or for hardware
work, and it runs N instances."

## Impact

Live migration keeps instances running (brief pause); cold migration and restarts stop them. The real impact is
decided per workload, and Nova cannot tell you which workload tolerates what. Collect that from the owning teams
before the campaign, as a table: instance or project, business criticality, acceptable downtime, contact.

## Quick checks

```bash
# 1. What runs on the host
openstack server list --all-projects --host <host> --long -c ID -c Name -c Status -c Flavor -c Project

# 2. Which instances cannot be live-migrated (PCI passthrough, SR-IOV direct ports, vGPU, special flavors)
for s in $(openstack server list --all-projects --host <host> -f value -c ID); do
  echo "== $s"; openstack server show $s -c flavor -c OS-EXT-SRV-ATTR:instance_name -f value
  for p in $(openstack port list --server $s -f value -c ID); do       # port list has no VNIC Type column
    echo "$p $(openstack port show $p -f value -c binding_vnic_type)"
  done | grep -v ' normal$'
done
# On the host: virsh dumpxml <instance-name> | grep -E 'hostdev|mdev'   -> passthrough or vGPU present

# 3. Capacity: where could they go?
openstack allocation candidate list --resource VCPU=<largest flavor vcpus> --resource MEMORY_MB=<largest flavor ram>
#    needs the osc-placement plugin. "openstack hypervisor list --long" no longer shows vCPUs and memory with recent
#    clients (python-openstackclient 9.0.0 on a lab): use "openstack resource provider usage show <rp-uuid>" instead

# 4. Version compatibility between source and destination
#    on each host: virsh version ; qemu-system-x86_64 --version ; the CPU model in nova.conf [libvirt] cpu_mode/cpu_models
```

## Diagnosis: choosing the operation per instance

| Instance | Operation | Why |
|---|---|---|
| Normal instance, shared storage or boot-from-volume | Live migration | Keeps it running; disks are not copied |
| Normal instance, local disk | Live block migration (`--block-migration`) | Copies the disk over the network: load on network and storage, longer |
| Memory-intensive, pages change faster than they copy | Live migration with auto-converge or post-copy enabled, else cold migration | Without them the migration never converges |
| PCI passthrough, SR-IOV direct port | Cold migration or restart | Live migration of these is unsupported or version-dependent |
| vGPU | Cold migration or restart on releases without mediated-device migration | Check your release |
| Workload the owner says may restart | Stop, patch, start, or cold migrate | Cheapest; needs no spare capacity |

**Direction matters.** Migration from a newer libvirt/QEMU to an older one is not supported. Once a host is
patched, instances can move onto it but not back off it to an unpatched host. The order of patching therefore
decides where instances can go: plan the sequence so that patched capacity is available to receive from the next
host. The destination's CPU model must also be compatible with the source's (`cpu_mode`, `cpu_models`).

## Possible causes

Why moving an instance off the host fails or stalls, and how to tell:

| Cause | How to tell |
|---|---|
| No destination with enough CPU, memory or disk | `No valid host` on the migration; `allocation candidate list` empty for the flavor |
| Destination CPU model incompatible | Source and destination `cpu_mode`/`cpu_models` differ. **Any:** with Nova's default, the pre-check refuses the move with `CPU doesn't have compatibility`. **Kolla-Ansible:** it writes `[workarounds] skip_cpu_compare_on_dest = true` (release note `skip-cpu-compare-on-dest`, 2025.1 and 2026.1), so Nova skips that comparison and libvirt checks the CPU on the destination while migrating: the migration fails after it starts, the instance stays on the source, and the source's `nova-compute` log has libvirt's error |
| Destination runs an older libvirt/QEMU than the source | Pre-check refuses the move; happens once a patching campaign has started |
| Memory changes faster than it is copied | Migration stays `running` with remaining memory not decreasing; no auto-converge or post-copy |
| Passthrough device, SR-IOV direct port, vGPU | `hostdev` or `mdev` in the domain XML; Nova refuses or the move fails late |
| Local disk without `--block-migration` | Error about shared storage; instance disks under the compute's instances path |

**Patching protects only restarted or migrated instances.** An instance that keeps running on the old QEMU binary
is not patched, even if the host package is. A live migration (which starts a new QEMU process on the destination)
or a restart is required for the fix to apply to the instance.

## Resolution

### Step 1: stop new placements on the host

> ⚠️ **State-changing.** No new instances land on the host; existing ones unaffected.

```bash
openstack compute service set --disable --disable-reason "maintenance <ticket> <date>" <host> nova-compute
```

### Step 2: live-migrate what can be live-migrated

> ⚠️ **State-changing, production impact.** The instance pauses briefly at switchover; network and storage carry the
> memory (and disk, for block migration) copy. On a lab, a ping lost between 1.8 and 8.6 s at the switch-over, and
> once the instance stayed unreachable for a few minutes after it (see Lab observations). Migrate a few at once, not
> the whole host, and watch the destination. Memory-heavy instances may never converge: set a timeout and be ready
> to abort.

```bash
# Let the scheduler pick a destination
openstack server migrate --live-migration <server-id>
# Or choose it (checked by the scheduler since Rocky unless --os-compute-api-version < 2.30)
openstack server migrate --live-migration --host <destination-host> <server-id>
# Local disks
openstack server migrate --live-migration --block-migration <server-id>

# Watch
openstack server migration list --server <server-id>
openstack server show <server-id> -c status -c OS-EXT-STS:task_state -c OS-EXT-SRV-ATTR:host
# Abort one that does not converge (migration id from the list above)
openstack --os-compute-api-version 2.24 server migration abort <server-id> <migration-id>
# Force completion: pauses the instance until the rest of its memory is copied, or switches to post-copy
# if it is enabled ([libvirt] live_migration_permit_post_copy = true). Agree the pause with the owner.
openstack --os-compute-api-version 2.22 server migration force complete <server-id> <migration-id>
```

See [live migration never completes](live-migration-never-completes-or-times-out.md) before forcing completion.

### Step 3: cold-migrate or restart the rest, as agreed with the owners

> 🛑 **Destructive for the workload's availability.** Cold migration stops the instance, moves it, and leaves it
> in `VERIFY_RESIZE` until confirmed; a restart interrupts the service. Precondition: the owner agreed to the
> downtime window. Undo: `openstack server resize revert` before confirming.

```bash
openstack server migrate <server-id>                 # cold migration
openstack server show <server-id> -c status          # VERIFY_RESIZE
openstack server resize confirm <server-id>          # newer clients: openstack server migration confirm
# or simply
openstack server stop <server-id>   # patch the host ... openstack server start <server-id>
```

### Step 4: patch and reboot the host, then re-enable

> ⚠️ **State-changing.** Re-enabling makes the host a destination again, which is exactly what the next host in the
> sequence needs.

After the reboot, wait until the host's `nova-compute` is `up` with a new `Updated At` in
`openstack compute service list --long`: for about a minute after the reboot it still shows `up`, from its last
heartbeat. On a lab, enabling a host still rebooting printed `Failed to set service status to enabled` (the API timed
out talking to the compute), although the change took effect.

```bash
openstack compute service set --enable <host> nova-compute
```

## Verification

```bash
openstack server list --all-projects --host <host>                      # empty before reboot
openstack compute service list --host <host>                            # up and enabled after
openstack server show <server-id> -c status -c OS-EXT-SRV-ATTR:host    # ACTIVE on the new host
openstack server migration list --server <server-id>                    # completed, none stuck in running
# On the destination: the instance's QEMU process is the new binary
ps -o pid,lstart,cmd -C qemu-system-x86_64 | grep <instance-name>
```

Check with the owner that the application is healthy after a cold migration or restart. For the campaign as a
whole: every host rebooted, every instance either migrated after its host was patched or restarted.

## Lab observations

On a lab: OpenStack 2026.1, Kolla-Ansible 22.2.0, 1 controller and 2 computes (nested KVM, local instance disks,
libvirt 10.0.0 and QEMU 8.2.2 on both), ML2/OVS, CirrOS instances, python-openstackclient 9.0.0.

| Step | Observed |
|---|---|
| Step 1, `--disable` | `Status disabled`, `State up`; a new instance went to the other compute |
| Quick checks 1, 2, 4 | As written; check 3 needed the changes above |
| Step 2, `openstack server migrate --live-migration --block-migration <server-id>` (and `--host <destination>`) | Each instance `ACTIVE` on the other compute after 32 to 34 s; `server migration list --server` showed `running` for a moment after, then `completed` |
| Connectivity during a live migration (ping every 0.2 s, five migrations) | Gaps of 1.8, 2.8, 3.1 and 8.6 s at the switch-over. Once, replies stopped at the switch-over and SSH failed for a few minutes; the instance had not restarted and answered again within about 5 minutes; the cause was not found |
| Step 4, reboot of the drained host, then `--enable` | See the note in step 4 |
| Verification | As written; on the destination, `ps -o pid,lstart,cmd -C qemu-system-x86_64` showed the instance's QEMU process started at the switch-over |

Not run: step 3 (cold migration, `resize confirm`), `migration abort` and `force complete`, instances with
passthrough devices, a version difference between hosts.

## Validity and limits

Derived from an operator's patching campaign on a platform of up to several hundred hosts, where spare capacity
did not allow live-migrating everything and the owning teams supplied the criticality that Nova cannot know.
Release not recorded. Command names for cold-migration confirmation changed across python-openstackclient
versions. Live migration constraints (PCI, SR-IOV, vGPU, CPU model, version direction) are release-dependent:
check the Nova documentation of your release. Capacity rule of thumb from the same source: spare capacity must
exist per failure domain (rack, availability zone), not once for the whole platform. Steps 1, 2 and 4 and the
verification were observed on a small lab on 2026.1 (see Lab observations).

## Related

- [Compute host failure: fence before evacuate](../recovery/compute-host-failure-fence-before-evacuate.md): never use evacuation on a host that may still be running its instances.
- [CPU steal on pinned instances](cpu-steal-on-pinned-instances.md): pinned instances need extra care when moved.
- Source: *OpenStack, the Day After Tomorrow* (S. Zaouam), chapter 20.

---

*Provenance: written by agent:ozarland-author, from the book *OpenStack, the Day After Tomorrow*.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*
