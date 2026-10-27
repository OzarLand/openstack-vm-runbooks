---
title: Instance goes SHUTOFF by itself, nobody stopped it. The compute's OOM killer killed its QEMU process
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [nova, libvirt, qemu, placement, linux-kernel]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: any
  storage: any
  os: linux
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-27, note: "cgroup-level OOM kill of one guest's QEMU (memory.max of its libvirt cgroup), not a host-wide OOM: quick checks, libvirt and nova-compute lines, Placement reads, server start, reserved_host_memory_mb override and reconfigure (nova_compute restarted, guest kept) run; undo by removing the override, not by explicit values; not run: host-wide OOM, cause 2 overcommit (ram_allocation_ratio), cause 3, the per-host nova.conf path" }
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#DEFAULT.reserved_host_memory_mb
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#DEFAULT.ram_allocation_ratio
  - https://docs.openstack.org/nova/2026.1/admin/scheduling.html
  - https://github.com/torvalds/linux/blob/master/mm/oom_kill.c
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Instance goes SHUTOFF by itself, nobody stopped it. The compute's OOM killer killed its QEMU process

## Symptom

An instance that was `ACTIVE` is `SHUTOFF`, and its owner did not stop it. The instance's actions show a `stop` that
nobody requested, made by Nova itself:

```text
$ openstack server event list <server-id>
| <request-id> | <server-id> | stop | <time> |
```

The guest's own logs end abruptly, with no shutdown sequence. Often, several large instances of the same compute are
affected at about the same time, or the same one again and again.

## Impact

The instance stopped without a clean shutdown: what was not written to disk is lost, and its file systems or databases
may need a recovery at the next boot. It stays `SHUTOFF` until someone starts it. While the compute is short of
memory, it can happen again to any instance on it, and the host's own services (Open vSwitch, `nova-compute`,
storage daemons) can be killed too.

## Quick checks

Read-only. On the instance's compute (`openstack server show <server-id> -c OS-EXT-SRV-ATTR:host -c
OS-EXT-SRV-ATTR:instance_name`):

```bash
# 1. Did the kernel kill a QEMU process? (dmesg -T for readable times)
sudo dmesg -T | grep -E 'invoked oom-killer|Killed process'
sudo journalctl -k --since "-2 days" | grep -E 'invoked oom-killer|Killed process'   # current boot only:
                                                    # add -b -1 for the previous boot (needs a persistent
                                                    # journal; journalctl --list-boots lists them)

# 2. The memory of the host now
free -m
```

The kernel's line names the process killed (`mm/oom_kill.c`):

```text
Out of memory: Killed process <pid> (qemu-system-x86) total-vm:<n>kB, anon-rss:<n>kB, file-rss:<n>kB, shmem-rss:<n>kB, UID:<uid> pgtables:<n>kB oom_score_adj:<n>
```

`Memory cgroup out of memory: Killed process ...` instead means a memory limit of a cgroup was reached, not the host's
memory; the kernel's `oom-kill:constraint=CONSTRAINT_MEMCG,...,oom_memcg=<cgroup>` line just before it names the
cgroup (for a guest of Kolla's `nova_libvirt`: `/machine/qemu-<n>-<instance-name>.libvirt-qemu`). The process name
may be shortened by the kernel (`qemu-system-x86`, `qemu-kvm`). The `<task> invoked oom-killer` line names the task
whose allocation triggered the kill, not necessarily the one killed.

Then the libvirt log of that guest, which records why QEMU stopped:

**Kolla-Ansible:** `sudo docker exec nova_libvirt tail -n 5 /var/log/libvirt/qemu/<instance-name>.log` (`podman exec`
with Podman). This directory is in the container's own layer: it is lost when `nova_libvirt` is recreated, so read it
before a redeploy.

**Any:** `sudo tail -n 5 /var/log/libvirt/qemu/<instance-name>.log`

```text
<timestamp>: shutting down, reason=crashed
```

And `nova-compute` on that compute, for the server ID:

**Kolla-Ansible:** `sudo grep <server-id> /var/log/kolla/nova/nova-compute.log | grep -E 'Lifecycle Event|shutdown by itself'`

**Any:** the same `grep` on the `nova-compute` log.

```text
VM Stopped (Lifecycle Event)
Instance shutdown by itself. Calling the stop API. Current vm_state: active, ...
```

## Diagnosis

### What Nova does when a guest dies

libvirt reports the guest as stopped. `nova-compute` logs `VM Stopped (Lifecycle Event)`, sees an instance `ACTIVE` in
its database whose guest is shut off, logs `Instance shutdown by itself. Calling the stop API.` and calls the stop API
(`nova/compute/manager.py`, `_sync_instance_power_state`). That is the `stop` in the instance's actions: Nova aligns
its record with the hypervisor, it does not restart the guest. If lifecycle events are missed, or turned off with
`[workarounds] handle_virt_lifecycle_events = false`, only the periodic power state sync does it, within
`sync_power_state_interval` (600 s by default).

So any guest that dies on the compute looks like this in Nova: a QEMU crash, a `kill`, or the OOM killer. The kernel
log tells which.

### Why the host runs out of memory

The scheduler places instances on a compute by its `MEMORY_MB` inventory in Placement: the host's memory, minus
`[DEFAULT] reserved_host_memory_mb` (default 512 MB), times `ram_allocation_ratio` (new computes start at
`initial_ram_allocation_ratio`, default 1.0). Kolla-Ansible 2026.1 does not change these for libvirt computes.

The host runs out of memory when what really runs there exceeds what the inventory assumed:

- the host's own processes use more than `reserved_host_memory_mb`: containers of the platform, Open vSwitch, a
  storage daemon (Ceph OSDs on a hyperconverged compute), monitoring agents, and memory the kernel cannot reclaim
  (shared memory and tmpfs, dirty pages under writeback, unreclaimable slab), whereas the page cache is reclaimed;
- QEMU uses more than the flavor's RAM for each guest (its own code, device emulation, I/O buffers);
- `ram_allocation_ratio` above 1.0 lets the guests' total exceed the host's memory, and the guests use it.

To compare, on the compute and in Placement:

```bash
free -m                                                         # what the host has and uses
openstack resource provider list --name <compute>               # its provider uuid
openstack resource provider inventory show <rp-uuid> MEMORY_MB  # total, reserved, allocation_ratio
openstack resource provider usage show <rp-uuid>                # MEMORY_MB allocated to instances
```

## Possible causes

### Cause 1: host processes larger than the reservation

`reserved_host_memory_mb` is left at 512 MB on a host whose own services use several GB. The scheduler fills the
compute up to its inventory, and the host has no room left.

### Cause 2: memory overcommit

`ram_allocation_ratio` above 1.0, and the guests use the memory they were given.

### Cause 3: something else grew on the host

A process outside OpenStack, a leak in a service, or a spike (a backup, a storage recovery) took the memory.

### Cause 4: not the OOM killer

No `Killed process` line for that time: QEMU stopped for another reason. The libvirt log of the guest and the
`nova_libvirt` log have it.

## Resolution

### Bring the instance back

After checking that the compute has free memory again (quick check 2), the owner or an admin starts it:

> ⚠️ **State-changing.** Starts the guest again. It boots like after a power loss: the guest may run file system or
> database recovery. Undo: `openstack server stop <server-id>`.

```bash
openstack server start <server-id>
```

If the compute is still short of memory, move instances away first, or the next one killed may be another
([maintenance](../migration/moving-instances-off-a-host-for-maintenance.md) for how to move them).

### For causes 1 and 2: make the inventory match the host

Read the compute's current inventory first, and write down `reserved` and `allocation_ratio`: they are the values to
restore if the change has to be undone.

```bash
openstack resource provider inventory show <rp-uuid> MEMORY_MB
```

Set `reserved_host_memory_mb` to what the host's own processes really use, with a margin. Set `ram_allocation_ratio`
only if you want to change the current ratio (to 1.0 unless the guests are known to leave memory unused): once set,
it overwrites any ratio set through the Placement API, and removing the option later does not put the old ratio back
(the option's help in Nova 2026.1).

**Kolla-Ansible:** in `/etc/kolla/config/nova/nova-compute.conf` (or per host, under
`/etc/kolla/config/nova/<inventory-hostname>/nova.conf`):

```ini
[DEFAULT]
reserved_host_memory_mb = <MB>
# only to change the ratio:
# ram_allocation_ratio = 1.0
```

> ⚠️ **State-changing.** `reconfigure`, or the restart of `nova-compute` without Kolla-Ansible, restarts
> `nova-compute` on the computes; running instances are not affected. The compute's `MEMORY_MB` inventory shrinks: the
> scheduler places fewer instances there. Instances already there stay, so a compute already over its new inventory
> stays over it until instances leave. Undo: set back what you changed to the values read above, explicitly
> (removing `ram_allocation_ratio` does not restore the ratio; if only `reserved_host_memory_mb` changed, there is no
> ratio to restore), and reconfigure or restart again.

```bash
kolla-ansible reconfigure -i <inventory> --tags nova --limit <computes>
```

**Any:** the same options in `nova.conf` on the computes, then restart `nova-compute`.

### For cause 3: find what grew

On the compute, sort processes by memory (`ps aux --sort=-rss | head`) and look at what the kernel's OOM report listed
just before the `Killed process` line (it prints the tasks and their memory). Fix or limit that process.

## Verification

```bash
openstack server show <server-id> -c status                      # ACTIVE
openstack resource provider inventory show <rp-uuid> MEMORY_MB   # reserved and allocation_ratio as set
```

On the compute, over the following days: no new `Killed process` line in `dmesg` or `journalctl -k`, and `free -m`
keeps a margin with every instance running.

## Lab observations

Kolla-Ansible 22.2.0 (2026.1) all-in-one, Ubuntu 24.04 (kernel 6.8), no swap, on 2026-10-27. Injected: a
cgroup-level OOM of one running `m1.tiny` CirrOS instance, by writing `50M` to `memory.max` of its cgroup
`/sys/fs/cgroup/machine/qemu-<n>-<instance-name>.libvirt-qemu` (QEMU used about 200 MB). Not a host-wide OOM.

- The kernel killed QEMU at once: `Memory cgroup out of memory: Killed process <pid> (qemu-system-x86)
  total-vm:2184636kB, anon-rss:177724kB, file-rss:8320kB, shmem-rss:0kB, UID:42436 pgtables:892kB oom_score_adj:0`,
  preceded by `memory: usage 180904kB, limit 51200kB, failcnt 41` and `oom-kill:constraint=CONSTRAINT_MEMCG,...`.
  Quick check 1 found it with both commands.
- The guest's libvirt log (read with `docker exec nova_libvirt tail`) ended with `shutting down, reason=crashed`,
  0.3 s after the write to `memory.max` (timestamps from the libvirt log and kernel captures).
- `nova-compute` logged `VM Stopped (Lifecycle Event)` 15 s after QEMU died, then `During _sync_instance_power_state
  the DB power_state (1) does not match the vm_power_state from the hypervisor (4)` and `Instance shutdown by itself.
  Calling the stop API. Current vm_state: active, current task_state: None, original DB power_state: 1, current VM
  power_state: 4`, then `Instance is already powered off in the hypervisor when stop is called.` (quoted from the
  `nova-compute` capture). The instance was `SHUTOFF`, and `openstack server event list` showed a `stop` whose
  request ID is the one of that log line.
- The SHUTOFF instance kept its `MEMORY_MB` allocation in Placement. `openstack server start`: `ACTIVE` in 18 s.
- `reserved_host_memory_mb = 4096` in `/etc/kolla/config/nova/nova-compute.conf` and `kolla-ansible reconfigure
  --tags nova --limit <compute>`: 4 min 42 s; `nova_compute` was recreated, `nova_libvirt` and the running guest's
  QEMU process were not; the inventory showed `reserved 4096`, `allocation_ratio 1.0`.
- Not run: a host-wide OOM (only a cgroup limit), cause 2 (`ram_allocation_ratio` not changed), cause 3, the
  per-host `<inventory-hostname>/nova.conf` path. The undo was done by removing the override and reconfiguring again
  (no ratio had been set), not by writing the old values explicitly.

## Validity and limits

- Written from Nova 2026.1 (`nova/compute/manager.py` `handle_lifecycle_event`, `_sync_instance_power_state`,
  `_stop_unexpected_shutdown_instance`; `reserved_host_memory_mb`, `ram_allocation_ratio`,
  `initial_ram_allocation_ratio`, `sync_power_state_interval`), Kolla-Ansible 2026.1 (`nova.conf.j2` of `nova-cell`),
  the Linux kernel (`mm/oom_kill.c`) and libvirt (`src/qemu/qemu_process.c`). Reproduced on an all-in-one lab with a
  cgroup-level OOM of one guest (*Lab observations*): the Nova, libvirt and kernel lines and the start and
  `reserved_host_memory_mb` resolutions. Not reproduced: a host-wide OOM, its `Out of memory: Killed process` line,
  and memory overcommit.
- An out-of-memory inside the guest is a different problem: the guest's own kernel kills a process, the instance stays
  `ACTIVE`.
- Not covered: hugepages and NUMA-pinned instances, memory ballooning, swap policy on computes, and `oom_score_adj`
  tuning. Hugepages change the arithmetic above: the libvirt driver reports `MEMORY_MB` `total` as the host's whole
  memory, with `reserved` only `reserved_host_memory_mb` (`update_provider_tree` in `nova/virt/libvirt/driver.py`), so a
  persistent hugepage pool stays in the inventory. Nova's huge pages guide says persistent huge pages are not usable by
  standard host processes and asks to reserve enough non-huge-page memory for them: without that, small-page guests
  can be placed into memory they cannot use.

## Related

- [No valid host was found, although capacity exists](../scheduling/no-valid-host-with-free-capacity.md): how Placement
  sees a compute's capacity.
- [Moving instances off a compute host](../migration/moving-instances-off-a-host-for-maintenance.md)
- [After a compute reboot, instances stay SHUTOFF](../recovery/after-a-compute-reboot-instances-shutoff-or-without-network.md):
  the other common reason for instances found `SHUTOFF`.
- Nova 2026.1: [configuration reference](https://docs.openstack.org/nova/2026.1/configuration/config.html),
  [scheduling](https://docs.openstack.org/nova/2026.1/admin/scheduling.html)

---

*Provenance: written by agent:ozarland-author, from the Nova and Kolla-Ansible 2026.1 documentation and source, and the
Linux kernel and libvirt source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=lifecycle/instance-shut-off-by-host-oom-killer.md&title=Feedback%3A+lifecycle/instance-shut-off-by-host-oom-killer.md):
your release, your deployment method, and what happened.
