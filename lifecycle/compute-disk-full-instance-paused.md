---
title: Compute disk full. An instance that writes is frozen, while Nova still shows it ACTIVE
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, libvirt, qemu]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers, 2 computes
  network: ml2/ovs (VXLAN)
  storage: local qcow2 instance disks on the computes, instance store on its own ext4 file system (no Cinder)
  os: ubuntu-24.04 (noble), docker
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible multinode (1 controller, 2 computes)", date: 2026-10-23, note: "full procedure on a disposable lab with a dedicated 5 GB instance store filled by a stray file (cause 3); power state returned to Running after the resume, and a new instance built afterwards; df on the Docker volume path needs sudo" }
upstream_refs:
  - https://docs.openstack.org/nova/latest/configuration/config.html#DEFAULT.disk_allocation_ratio
  - https://docs.openstack.org/nova/latest/configuration/config.html#DEFAULT.reserved_host_disk_mb
  - https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/compute-service.html
last_verified: 2026-10-23
---

# Compute disk full. An instance that writes is frozen, while Nova still shows it ACTIVE

A full compute disk is silent until an instance writes. Then QEMU pauses that instance, Nova keeps it `ACTIVE`, and
the guest reports no error: its write just stops. Nothing else on the platform notices.

## Symptom

At first, none. The file system that holds the instances of a compute is full, and Nova goes on building instances
on it: a new instance from a cached image needs almost no space at first.

Later, a user reports that an instance "hangs": it stops answering, or a process inside it never finishes writing.
In Nova:

```text
$ openstack server show <server-id> -c status -c OS-EXT-STS:power_state -c OS-EXT-SRV-ATTR:host
| OS-EXT-STS:power_state | Paused    |
| OS-EXT-SRV-ATTR:host   | compute-1 |
| status                 | ACTIVE    |
```

On the compute, libvirt gives the reason:

```text
$ docker exec nova_libvirt virsh domstate <domain> --reason
paused (I/O error)
```

The guest console shows the last thing it printed before the write, and nothing after: no error, no end of the
write.

## Impact

- Instances of that compute that write to their disk freeze, one after the other. Their data is kept: a paused
  instance loses nothing, it waits.
- Instances of the same compute that do not write keep running and serving.
- Nova and Placement do not report anything. Placement counts the disk the instances were given (their flavors), not
  the space they use, so the scheduler keeps sending new instances to the full compute.
- The power state (Paused) is the only signal in Nova.

## Quick checks

On a controller or the deploy host, with admin credentials:

```bash
# 1. Instances that Nova shows ACTIVE but paused
openstack server list --all-projects --long -c ID -c Name -c Status -c "Power State" -c Host | grep -i paused
```

On the compute:

**Kolla-Ansible:**

```bash
# 2. Is the instance store full? (default store: the data of the nova_compute volume)
df -h /var/lib/docker/volumes/nova_compute/_data /

# 3. Which domains are paused, and why
for d in $(docker exec nova_libvirt virsh list --all --name); do
  echo "$d: $(docker exec nova_libvirt virsh domstate "$d" --reason)"
done
```

**Any:**

```bash
df -h /var/lib/nova/instances /
for d in $(virsh list --all --name); do echo "$d: $(virsh domstate "$d" --reason)"; done
```

This page applies when the file system of the instance store is at 100 % and at least one domain is
`paused (I/O error)`. A domain paused on an I/O error while the store is not full is another problem (another
storage path, or another cause of the I/O error). A full store with no paused domain is this problem before it
shows: act now, nothing has frozen yet.

## Diagnosis

### Where the instance store is

**Kolla-Ansible:** the instances live in the `nova_compute` Docker volume (`nova_instance_datadir_volume`), by
default `/var/lib/docker/volumes/nova_compute/_data` on the host, `/var/lib/nova` inside the container. Check
whether it is a file system of its own or shares `/`:

```bash
findmnt -T /var/lib/docker/volumes/nova_compute/_data
```

If it shares the root file system, a full store is also a full `/`, and the whole host is affected (logs, Docker,
other containers).

### What filled it

Read-only, on the compute. Find the cause before deleting anything.

```bash
# Kolla-Ansible paths; on other deployments use /var/lib/nova
cd /var/lib/docker/volumes/nova_compute/_data
du -xsh * 2>/dev/null | sort -h | tail
du -xsh instances/* 2>/dev/null | sort -h | tail       # per-instance directories, and _base
ls -la instances/_base | head
```

Typical consumers: instance disks that grew (qcow2 overlays grow as guests write), cached base images in
`instances/_base`, logs or core dumps, or a stray file left by someone.

Map a directory or a domain to its instance: the directory name under `instances/` is the instance ID, and
`virsh domuuid <domain>` gives the same ID.

### Nova and logs

```bash
# On the compute: nothing is expected here, which is part of the signature
grep -iE 'No space|ENOSPC' /var/log/kolla/nova/nova-compute.log | tail -n 5
```

## Possible causes

| Cause | How to tell |
|---|---|
| Instance disks grew beyond what the store can hold (overcommitted local disk) | `du` shows the space in the instance directories; the sum of flavors' disks exceeds the store |
| Base image cache | `instances/_base` holds a large share |
| Something else on the same file system (logs, images, a stray file) | `du` at the top of the store, or of `/` when the store shares it |

All three end the same way for the instances; the fix differs only in what you free.

## Resolution

### Step 1: stop placing instances on this compute

> ⚠️ **State-changing.** The scheduler stops sending new instances to this compute. Running instances are not
> affected. Undo: `openstack compute service set --enable <compute-host> nova-compute`.

On a controller (**Any:**):

```bash
openstack compute service set --disable --disable-reason "instance store full <date>" <compute-host> nova-compute
```

### Step 2: free space

Decide what to remove from what the diagnosis found. Never delete or truncate an instance disk, and never remove a
file from `instances/_base` that a running instance still uses as its backing file.

> 🛑 **Destructive.** Deleting a file cannot be undone. A wrong file can be an instance disk or the base image an
> instance's disk depends on, and that instance would lose its data or fail to restart. Affects whatever the file
> belonged to. Precondition: you know what the file is (`du`, `ls -la`, `qemu-img info <file>` for disk images) and
> that nothing uses it. Back it up elsewhere first if in doubt. To check whether a running QEMU holds a file open:
> `fuser <file>` or `lsof <file>`.

```bash
# On the compute, for a file you have identified as safe to remove
sudo rm <file>
```

If the space is used by the instances themselves, deleting files is not the fix: move instances off the compute
(see [Moving instances off a host](../migration/moving-instances-off-a-host-for-maintenance.md)) or extend the file
system. A paused instance needs its space back on this compute before it can move.

### Step 3: resume the paused instances

Freeing space does not resume them. QEMU paused them on the failed write, and they stay paused until resumed.
`openstack server unpause` does not apply: Nova sees them `ACTIVE`, not `PAUSED`.

> ⚠️ **State-changing.** Resumes the guest where it stopped; it retries the write. Affects this instance only.
> Precondition: free space on the store, or the instance pauses again at its next write. Undo:
> `virsh suspend <domain>`.

On the compute:

**Kolla-Ansible:**

```bash
docker exec nova_libvirt virsh resume <domain>

# Or every paused domain of the compute
for d in $(docker exec nova_libvirt virsh list --state-paused --name); do
  docker exec nova_libvirt virsh resume "$d"
done
```

**Any:**

```bash
virsh resume <domain>
```

### Step 4: keep it from coming back

Not tested on the lab. Options to weigh:

- Monitor the actual usage of the instance store on every compute, and the power state of instances (Paused while
  `ACTIVE`). Nova and Placement do not watch the disk.
- Reserve disk for the host with `reserved_host_disk_mb`, and lower `disk_allocation_ratio` if local disks are
  overcommitted. Both change what Placement offers, not what the guests write.
- Put the instance store on a file system of its own, so that a full store does not fill `/`.

When space is back and the cause is handled, enable the compute service again:

> ⚠️ **State-changing.** The scheduler may place new instances on this compute again.

```bash
openstack compute service set --enable <compute-host> nova-compute
```

## Verification

On the compute:

```bash
df -h /var/lib/docker/volumes/nova_compute/_data           # below 100 %, with margin
docker exec nova_libvirt virsh list --state-paused --name    # empty
```

On a controller:

```bash
openstack server list --all-projects --host <compute-host> --long -c ID -c Status -c "Power State"
```

Expected: no domain paused, store below 100 %, instances `ACTIVE` and Running. Check inside a resumed instance that
its interrupted work finished or failed cleanly. The lab did not check that a new instance on the compute becomes
`ACTIVE` after the fix; check it separately if you need that certainty.

## Lab observations

On the lab, the instance store of compute-1 was filled with one file, leaving about 50 MiB free, and a test
instance that writes 500 MB at boot was then placed on that compute.

First runs (2026-10-09), when the instance store was still on the root file system of the compute:

| First run | Before | During (180 s) | After (+60 s) |
|---|---|---|---|
| File system of compute-1 (root, holding the store) | 17 % used, 40 GB free | **100 %, 21 MB free** | 17 % used |
| Test instance on compute-1 (writes almost nothing) | none | **ACTIVE in 17 s** | deleted |
| Instances in libvirt on compute-1 | 1 running | **2 running**, none paused | 1 running |
| Requests to the running instances | – | **377 each, 0 lost** | 564, 0 lost |
| `nova-compute.log` (`No space`, `ENOSPC`) | nothing | **nothing** | nothing |

| Second run (test instance writes 500 MB at boot) | Before | During (240 s) | After (fix + 60 s) |
|---|---|---|---|
| File system of compute-1 (root, holding the store) | 17 % used | **100 %, 4.5 MB free** | 17 % used |
| Test instance in libvirt | none | **paused (I/O error)** | resumed, then deleted |
| Test instance in Nova | none | **ACTIVE, power state 3 (Paused)** | deleted |
| Test instance console | – | `writing to the disk`, then nothing: no error, no end of the write | – |
| Other instance on compute-1 (not writing) | running | **running, 503 requests, 0 lost** | running |

Two later builds (2026-10-13), with the instance store on an ext4 volume of its own (9.8 GiB) mounted on the data
directory of `nova_compute`:

- During the failure, on both builds: the store at 100 % (456 KiB free) while `/` stayed at 17 %; 1 of the 2
  domains of compute-1 `paused (I/O error)`; the test instance `ACTIVE` in Nova with power state 3 (Paused).
- Before and after: the store at 1 %, `/` at 17 %, the one domain of compute-1 not paused.
- Fix observed: removing the fill file, then `virsh resume` of the paused domains. Freeing the space alone did not
  resume the paused instance; `virsh resume` did.
- A new instance on the full compute became `ACTIVE`: the scheduler and Placement did not see the full disk.

## Validity and limits

- Observed on four lab builds with the configuration above: OpenStack 2026.1, Kolla-Ansible 22.2.0, ml2/OVS, Docker,
  Ubuntu 24.04, 3 controllers and 2 computes, local qcow2 disks, CirrOS guests, no Cinder. Two runs had the store on
  the root file system, two on a dedicated ext4 file system.
- Not tested: OVN, other releases, other deployment tools, production scale, Ceph or other shared instance storage,
  volume-backed instances, raw or LVM image backends, guests other than CirrOS.
- Observed resolution steps: freeing space (removing a known file) and `virsh resume`. Not observed: disabling the
  compute service, finding the cause on a real store, removing base images, Step 4, and whether Nova's power state
  returns to Running after a resume (the lab deleted the test instance right after).
- The pause comes from QEMU's default write-error policy (`werror=enospc`); Nova sets no `error_policy` in the domain
  XML on this release. Not checked on other releases.

## Related

- [nova-compute down, instances still running](../recovery/nova-compute-down-instances-still-running.md): a domain
  paused on an I/O error still holds its QEMU process and disk locks; never evacuate it without fencing.
- [Moving instances off a host](../migration/moving-instances-off-a-host-for-maintenance.md)
- [No valid host with free capacity](../scheduling/no-valid-host-with-free-capacity.md): the other side of the
  scheduler's view of capacity.
- Nova configuration: [`disk_allocation_ratio`](https://docs.openstack.org/nova/latest/configuration/config.html#DEFAULT.disk_allocation_ratio),
  [`reserved_host_disk_mb`](https://docs.openstack.org/nova/latest/configuration/config.html#DEFAULT.reserved_host_disk_mb).
- Source: lab reproduction by the OzarLand maintainer (OpenStack 2026.1, Kolla-Ansible 22.2.0).
