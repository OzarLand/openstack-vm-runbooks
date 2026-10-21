---
title: Compute host unreachable, instances down. Fence before you evacuate
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, libvirt, cinder, ceph]
deployment_methods: [any]
environment:
  topology: any multinode
  network: any
  storage: shared instance storage or boot-from-volume for evacuation to preserve disks
  os: any
risk: destructive
confirmed_by:
  - { who: "lab:ozarland-maintainer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, 3 controllers + 2 computes, OVS, Docker, Ubuntu Noble", date: 2026-10-13, note: "failure (abrupt power-off of a compute) injected and observed on 3 lab builds; symptom observed (compute service and OVS agent down, instance still ACTIVE on the dead host in Nova); service marked down, evacuation (microversion 2.29, local disk, rebuilt from image) and cleanup by nova-compute on the host's return observed; fencing through IPMI or the switch not run (the host was off by construction), disabling the service, Cinder and Ceph not tested" }
upstream_refs:
  - https://docs.openstack.org/nova/latest/admin/evacuate.html
  - https://docs.openstack.org/nova/latest/admin/node-down.html
last_verified: 2026-10-21
---

# Compute host unreachable, instances down. Fence before you evacuate

`nova evacuate` rebuilds an instance on another host and reuses its volumes. Nova requires the source compute
service to be reported `down` first, but Nova cannot know whether the host is really dead or merely unreachable
from the control plane. Evacuating from a host that is still running the instances produces two copies of each
server writing to the same volume. The precondition is fencing, not the missing heartbeat.

## Symptom

```text
$ openstack compute service list --service nova-compute -c Host -c State
| compute-07 | down |

$ openstack server list --all-projects --host compute-07 -c ID -c Status
| ...        | ACTIVE |      <- Nova still believes they run; users say they are unreachable
```

Monitoring shows the host unreachable (no ping, no SSH, no agent). Users report their instances on that host are
not responding.

Nova keeps those instances `ACTIVE`, power state Running, on the dead host: it cannot know they stopped. The
host's Neutron agent is usually reported dead too, but not necessarily at the same moment: Neutron's
`agent_down_time` (75 s by default) can lag Nova's `service_down_time` (60 s), so for a while the agent may still
show as alive.

## Impact

Every instance on the host is down or unreachable. Nothing else is affected unless the host also ran a network
node role. The risk during recovery is worse than the outage: a wrong evacuation corrupts volumes.

## Quick checks

```bash
# 1. Nova's view of the host and when it last reported
openstack compute service list --service nova-compute --host <host> --long

# 2. Which instances and which volumes are involved
openstack server list --all-projects --host <host> --long -c ID -c Name -c Status -c Networks
openstack volume list --all-projects --status in-use -c ID -c "Attached to" | grep -F "<instance-id>"

# 3. Can the control plane reach the host at all?
ping -c 3 <host>             # management network
ssh <host> uptime            # if it answers, the host is NOT dead; stop here and diagnose the service instead
```

If the host answers on the management network, this is a `nova-compute` or message-bus problem, not a dead host.
Do not evacuate. Restart `nova-compute` and investigate its connection to RabbitMQ: see
[nova-compute down, instances still running](nova-compute-down-instances-still-running.md). If the host answers on
TCP but refuses your SSH login, it is alive too: see
[nova-compute down, host alive, SSH refused](nova-compute-down-host-alive-ssh-refused.md). Only a failure at the TCP
level, confirmed out of band, points to a dead host.

## Diagnosis

The question to answer is "is the host powered off or isolated?", and the platform cannot answer it. Use a channel
that does not depend on the host's network stack:

- **Out-of-band management** (IPMI, iDRAC, iLO, BMC): power state, console. This is the authoritative source.
- **Switch side**: is the host's port up, is traffic flowing? A host that is up with a dead management NIC may still
  be running instances and serving them on the data-plane network.
- **Storage side**: with Ceph, `ceph osd blocklist ls` and client sessions can show whether the host's libvirt
  clients still hold RBD images open (`rbd status <pool>/<image>` lists watchers).

A host that is "up" by any of these channels is isolated, not dead. Instances may still be running on it.

## Possible causes

| Cause | How to tell |
|---|---|
| Hardware failure, host powered off | Out-of-band interface shows powered off or a hardware fault |
| Kernel panic or hang | Console shows a panic; power state is on |
| Management network isolation | Host reachable from out-of-band console; instances still running; data plane may be fine |
| `nova-compute` crashed or lost its bus connection | Host reachable over SSH; only the service is down |

## Resolution

### Step 1: fence

> 🛑 **Destructive.** Powering off the host ends every instance still running on it, if any. It is the only way to
> make evacuation safe. Affects all instances on this host. Precondition: you have decided the host must be taken
> out; the instances are already unavailable to users or the risk of split-brain outweighs the remaining service.
> Cannot be undone, but the instances will be rebuilt on other hosts in step 3.

Power off through the out-of-band interface and confirm the power state:

```bash
# -E reads the password from the IPMI_PASSWORD environment variable, so it stays out of the process list and history
ipmitool -I lanplus -H <bmc-address> -U <user> -E chassis power off
ipmitool -I lanplus -H <bmc-address> -U <user> -E chassis power status
```

If out-of-band management is unavailable, isolate the host at the switch (shut its ports) so it cannot write to
storage or serve traffic. Isolation at the switch is weaker than power-off (the host may still write to local
disks) but prevents the double-writer problem on shared storage.

### Step 2: keep the scheduler away from the host

> ⚠️ **State-changing.** Prevents new instances from being placed on the host when it returns.

```bash
openstack compute service set --disable --disable-reason "fenced <date>, hardware failure" <host> nova-compute
```

Optionally, mark the service down so that Nova accepts the evacuation without waiting for `service_down_time`. Do
it only after step 1: it tells Nova the host is fenced.

> ⚠️ **State-changing.** Nova treats the compute service as down at once, whatever its heartbeat says. Affects this
> host only. Needs compute API microversion 2.11 or later. Undo: `openstack compute service set --up <host>
> nova-compute` (step 4).

```bash
openstack compute service set --down <host> nova-compute
```

### Step 3: evacuate

> 🛑 **Destructive.** Evacuation rebuilds the instance on another host. With shared instance storage or
> boot-from-volume, the disk is preserved. With local ephemeral storage, the instance is rebuilt from its image and
> **the ephemeral disk content is lost**. Check each instance's storage before evacuating. Precondition: step 1
> done; Nova reports the compute service `down`.

```bash
# One instance; omit --host to let the scheduler choose
openstack server evacuate --host <destination-host> <server-id>
# Older clients: nova evacuate [--on-shared-storage] <server-id> [<destination-host>]
```

From compute API microversion 2.95, an evacuated instance stays stopped on its new host and must be started
(`openstack server start <server-id>`); before 2.95 it comes back `ACTIVE`. Check which microversion your client
sends, or pass it explicitly with `--os-compute-api-version`.

To evacuate every instance of the host in one go, check whether your deployment provides `nova host-evacuate`
(legacy client) or script a loop over the list from the quick checks. Evacuate in batches on a large host: each
rebuild loads the destination's storage and network.

Automation that evacuates on a lost heartbeat (Masakari, Pacemaker fence agents, home-made scripts) must perform
step 1 itself, or it will one day destroy the workload it was meant to save.

### Step 4: when the host comes back

Nova handles a returning evacuated host: on startup, `nova-compute` finds the evacuation migration records and
destroys the local copies of the evacuated instances. Do not start `nova-compute` on the host before confirming the
evacuations completed, and keep the service disabled until the host is diagnosed.

If you marked the service down in step 2, mark it up once the host is back and `nova-compute` reports again (its
`Updated At` changes in `openstack compute service list --long`):

> ⚠️ **State-changing.** Nova reads the service state from its heartbeat again. Precondition: the evacuations are
> complete and the host's `nova-compute` has started and cleaned up. Undo: `--down` again.

```bash
openstack compute service set --up <host> nova-compute
```

Then check on the host that libvirt no longer holds the evacuated instances:

```bash
# On the host. Kolla-Ansible:
docker exec nova_libvirt virsh list --all
# Any:
virsh list --all
```

## Verification

```bash
# Every instance has a new host and is ACTIVE (SHUTOFF until started, from microversion 2.95)
openstack server list --all-projects --long -c ID -c Status -c Host | grep -F "<previous-host>"   # should be empty
openstack server show <server-id> -c status -c OS-EXT-SRV-ATTR:host

# Volumes are attached to the rebuilt instances, once
openstack volume show <volume-id> -c status -c attachments

# Migration records
openstack server migration list --type evacuation --status done

# The source host is still disabled
openstack compute service list --host <previous-host>
```

Reach a sample of the evacuated instances from the network and check their application state; a rebuilt instance
boots from scratch and may need its services started.

## Lab observations

On the lab, compute-2, a virtual machine, was powered off at once with no clean shutdown, as a power failure would,
and stayed off until the recovery powered it on again through its hosting cloud: it was fenced by construction.

| | Before | During (120 s) | After evacuation and return of the host |
|---|---|---|---|
| Instance on compute-2, seen from the network | answering | **down** (from 2 s after the power-off) | answering again after a gap of **181.5 s** |
| Instance on compute-1, seen from the network | answering | 291 requests, 0 lost | 1155 requests, 0 lost |
| Instance of compute-2 in Nova | ACTIVE on compute-2 | **ACTIVE, running, on compute-2** (host off) | evacuated: ACTIVE on compute-1 in about 15 s |
| Services | 22 up | **nova-compute and Open vSwitch agent of compute-2 down** | 22 up |
| libvirt on compute-2 after its return | – | – | **empty**: the evacuated instance was cleaned up |

- Recovery as run on the lab: `openstack compute service set --down`, then `openstack server evacuate <server-id>`
  with `--os-compute-api-version 2.29` and no destination (the scheduler chose compute-1), then power-on of the host,
  wait until its `nova-compute` reported again (a new `Updated At`), `openstack compute service set --up`, and
  `virsh list --all` on the host.
- The 181.5 s include the 120 s the lab waited before recovering (in production: detection and decision), the
  evacuation (about 15 s) and the boot of the instance on compute-1 with its floating IP (about 45 s).
- The instance had a local disk: it was rebuilt from its image on compute-1, and kept its port and floating IP.
- On the host's return, `nova-compute` destroyed its copy of the evacuated instance: libvirt on compute-2 was empty.
- Instances on the other compute were not affected.
- Same result on three lab builds; last observed run 2026-10-13.

## Validity and limits

Derived from a real incident described by an operator: the team refused to evacuate on the strength of a missing
heartbeat, confirmed the host's state through its out-of-band interface, shut it down, and only then evacuated.
The commands assume the libvirt driver. Evacuation semantics for local storage, and the behaviour of a returning
host, should be checked against the Nova documentation of your release.

The lab part was observed on lab builds with OpenStack 2026.1, Kolla-Ansible 22.2.0, ml2/OVS, Docker, Ubuntu 24.04,
3 controllers and 2 computes, local instance disks and no Cinder. Observed: the symptom, marking the service down,
the evacuation of one instance with a local disk (microversion 2.29), and the cleanup on the host's return with the
service marked up. Not observed on the lab: fencing through IPMI or the switch (the host was off by construction),
disabling the service (step 2), evacuation with shared storage, boot from volume, Cinder or Ceph, the Ceph checks of
the diagnosis, evacuation in batches, microversion 2.95 or later, OVN, other releases and deployment tools,
production scale.

## Related

- [Moving instances off a host for planned maintenance](../migration/moving-instances-off-a-host-for-maintenance.md): the planned counterpart, where the host is healthy.
- [Stale records](../lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md): what an interrupted evacuation leaves behind.
- [nova-compute down, instances still running](nova-compute-down-instances-still-running.md): the host is alive,
  only the agent is down. Do not evacuate.
- [nova-compute down, host alive, SSH refused](nova-compute-down-host-alive-ssh-refused.md): a refused login is not
  a dead host.
- Source: *OpenStack, the Day After Tomorrow* (S. Zaouam), chapter 10, "Unplanned host failure".
- Source: lab reproduction by the OzarLand maintainer (OpenStack 2026.1, Kolla-Ansible 22.2.0).
