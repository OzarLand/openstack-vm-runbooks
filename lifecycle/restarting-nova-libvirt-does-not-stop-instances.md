---
title: I need to restart nova_libvirt on a compute. Its instances keep running
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, libvirt]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers, 2 computes
  network: ml2/ovs (VXLAN)
  storage: local ephemeral disk on the computes, no Cinder
  os: ubuntu-24.04 (noble), docker
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-22, note: "full procedure on a disposable lab, quick checks 2 and 3 and the qemu PID comparison included; a stop requested during the restart completed; stuck libvirtd and live migration not tested" }
  - { who: "@soufian-zaouam", type: operator, release: "2024.x", deployment: "kolla-ansible, production, multinode", date: 2026-10-24, note: "behaviour observed in the owner's production operations on Kolla-Ansible 2024.x or earlier; matches the page; the page's steps were not re-run from this text" }
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/latest/reference/compute/libvirt-guide.html
  - https://docs.openstack.org/nova/latest/admin/configuration/hypervisor-kvm.html
authored_by: agent:ozarland-author
last_verified: 2026-10-22
---

# I need to restart nova_libvirt on a compute. Its instances keep running

A planned operation, not a failure: you must restart `nova_libvirt` on a compute (a configuration change, a stuck
libvirtd, or a Nova reconfigure or upgrade, which restarts it on every compute). This page says what happens to the
instances, what to check first and how to verify afterwards.

## Symptom

"I need to restart `nova_libvirt` on `<compute>`, which runs instances. Will they stop?"

With Kolla-Ansible's default settings, on the lab, they did not: same domains, no request lost.

## Impact

- **Running instances:** they keep running. Their `qemu` processes are host processes, not children confined in the
  container; the new libvirtd finds them and takes them back.
- **Libvirt operations on that compute wait** until libvirtd is back: start, stop, reboot, new instances, live
  migrations to or from the compute. nova-compute talks to libvirt and cannot act meanwhile.
- **A live migration in progress** during the restart was not tested. Do not restart while one runs.
- **If `nova_libvirt` does not run in the host's PID namespace** (a changed deployment), a restart ends the qemu
  processes with the container: every instance of the compute stops. Quick check 3 rules this out.

## Quick checks

Before the restart. On a controller, with admin credentials:

```bash
# 1. What runs on the compute, and whether anything is in progress
openstack server list --all-projects --host <compute> --long -c ID -c Name -c Status -c "Task State" -c "Power State"

# 2. No migration in progress to or from the compute
openstack server migration list --host <compute> --status running
```

On the compute:

```bash
# 3. nova_libvirt runs in the host's PID and cgroup namespaces: both must print "host"
docker inspect -f '{{.HostConfig.PidMode}} {{.HostConfig.CgroupnsMode}}' nova_libvirt

# 4. Record the domains and the qemu processes, to compare after the restart
docker exec nova_libvirt virsh list --all
pgrep -a -f qemu-system
```

If Quick check 3 does not print `host host`, **do not restart**: the instances would stop. Move them off the compute
first (see Related).

## Diagnosis

### Why the instances survive

Kolla-Ansible runs `nova_libvirt` with `pid_mode: host` and `cgroupns_mode: host`. A qemu process started by
libvirtd is then a process of the host, in the host's cgroups, and it outlives the container. When the container
stops, the processes stay; when it starts again, libvirtd reads its state and re-attaches to the running domains. It
does not start them again: the domain IDs stay the same.

### Libvirt

To follow the restart, on the compute:

```bash
docker logs --tail 50 -f nova_libvirt
```

### Nova

nova-compute reconnects to libvirt by itself. If it logs connection errors that do not stop once libvirtd is back,
read `/var/log/kolla/nova/nova-compute.log`.

## Possible causes

This page has no failure cause. The reasons to restart libvirt are:

### Cause 1: planned restart

A configuration change, a Nova reconfigure or upgrade. Follow the resolution below.

### Cause 2: libvirtd is stuck

`docker exec nova_libvirt virsh list` hangs, and nova-compute logs libvirt timeouts. A restart is the usual first
step; this case itself was not reproduced on the lab, only the restart of a healthy libvirtd.

## Resolution

### Restart nova_libvirt

On the compute, after Quick checks 2 and 3.

> ⚠️ **State-changing.** Restarts libvirtd on this compute. Running instances kept running on the lab, but libvirt
> operations on this compute (start, stop, new instances, migrations) wait until libvirtd is back. Precondition:
> Quick check 3 prints `host host` and no migration runs; otherwise every instance of the compute can stop or a
> migration can fail. Undo: none needed; if the container does not come back, start its unit.

**Kolla-Ansible:**

```bash
systemctl restart kolla-nova_libvirt-container.service
```

**Any:** with a packaged libvirt, the service is `libvirtd` (or the modular `virtqemud`). Whether instances survive
its restart there was not tested on the lab.

```bash
systemctl restart libvirtd
```

If the container does not come back:

> ⚠️ **State-changing.** Starts libvirtd on this compute. Affects libvirt operations on this compute only. Undo: stop
> the same unit (instances survive it only under the precondition above).

```bash
systemctl start kolla-nova_libvirt-container.service
```

## Verification

On the compute:

```bash
# Same domains, same IDs, as recorded in Quick check 4
docker exec nova_libvirt virsh list --all

# Same qemu PIDs
pgrep -a -f qemu-system
```

On a controller:

```bash
openstack server list --all-projects --host <compute> --long -c ID -c Name -c Status -c "Power State"
openstack compute service list --service nova-compute --host <compute>
ping -c 10 <instance-address>
```

Expected: the same domain IDs, the same qemu PIDs, instances `ACTIVE` and `Running`, nova-compute `up`, pings
answered.

To measure the interruption yourself, start `ping -i 0.5 -D <instance-address>` from a host that reaches the
instance before the restart and read the sequence numbers afterwards.

## Lab observations

On the lab, the systemd unit of `nova_libvirt` was restarted on one compute, as a reconfigure or upgrade does. A
request was sent every 0.5 s to the instance on that compute.

| | Before | During (60 s after the restart) | After (+60 s) |
|---|---|---|---|
| Instance on compute-1 | no loss | **174 requests, 0 lost** | 339 requests, 0 lost |
| `virsh list` on compute-1 | domain 1, running | **domain 1, running** | domain 1, running |
| Instance state in Nova | ACTIVE, running | ACTIVE, running | ACTIVE, running |

- The domain kept ID 1: libvirt took it back, it did not start it again.
- The qemu PID was recorded before the restart only. The unchanged domain ID and the absence of loss show the
  process was not started again; the PID after the restart was not compared.

## Validity and limits

- Observed on one lab build with the configuration above: OpenStack 2026.1, Kolla-Ansible 22.2.0 with its default
  `nova_libvirt` settings (`pid_mode: host`), Docker, Ubuntu 24.04, one instance on the restarted compute.
- Also observed in production, on a multinode Kolla-Ansible platform on an earlier release (2024.x or earlier,
  exact release not recorded), by @soufian-zaouam, who confirmed on 2026-10-24 that the page matches; the page's
  steps were not re-run there.
- Observed: the restart of the unit, no loss, same domain ID, instance `ACTIVE` and running.
- Not run on that build: Quick checks 2 and 3 (the lab used defaults), the PID comparison after the restart, a
  libvirt operation during the restart. A lab review on 2026-10-22 (all-in-one, see `confirmed_by`) ran the full
  procedure with Quick checks 2 and 3 and the qemu PID comparison, and a stop requested during the restart
  completed.
- Not run on any lab: the restart of a stuck libvirtd (cause 2), a live migration during the restart. The `Any:`
  commands are not recorded as run.
- Not tested: a deployment with a non-host PID namespace (expected to stop the instances, not observed), Podman,
  other releases, other deployment tools, production scale, instances with Cinder volumes, PCI passthrough or vGPU.
- An interruption shorter than 0.5 s could have been missed.

## Related

- [Restarting the Open vSwitch agent does not cut instances](../networking/restarting-the-ovs-agent-does-not-cut-instances.md):
  the same question for the network agent.
- [nova-compute down, instances still running](../recovery/nova-compute-down-instances-still-running.md)
- [Moving instances off a host for maintenance](../migration/moving-instances-off-a-host-for-maintenance.md): when
  the precondition does not hold.
- Kolla-Ansible, libvirt guide: https://docs.openstack.org/kolla-ansible/latest/reference/compute/libvirt-guide.html
- Nova, KVM hypervisor: https://docs.openstack.org/nova/latest/admin/configuration/hypervisor-kvm.html
- Source: lab reproduction by @soufian-zaouam (OpenStack 2026.1, Kolla-Ansible 22.2.0).

---

*Provenance: written by agent:ozarland-author, from a lab reproduction by @soufian-zaouam on 2026.1.
Reviewed in its pull request (see the page history).
Checked by a human operator: not yet.
Behaviour also reported from production by @soufian-zaouam (Kolla-Ansible 2024.x or earlier, steps not re-run).*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=lifecycle/restarting-nova-libvirt-does-not-stop-instances.md&title=Feedback%3A+lifecycle/restarting-nova-libvirt-does-not-stop-instances.md):
your release, your deployment method, and what happened.
