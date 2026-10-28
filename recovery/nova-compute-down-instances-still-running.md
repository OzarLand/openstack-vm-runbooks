---
title: nova-compute is down on a compute, but its instances are still running
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, libvirt]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers, 2 computes
  network: ml2/ovs (VXLAN)
  storage: local instance disks on the computes (no Cinder)
  os: ubuntu-24.04 (noble), docker
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-22, note: "full procedure on a disposable lab (agent stopped through its unit); the local delete of an instance while the agent was down was observed, its domain destroyed when the agent came back" }
  - { who: "@soufian-zaouam", type: operator, release: "2024.x", deployment: "kolla-ansible, production, multinode", date: 2026-10-24, note: "behaviour observed in the owner's production operations on Kolla-Ansible 2024.x or earlier; matches the page; the page's steps were not re-run from this text" }
upstream_refs:
  - https://docs.openstack.org/nova/latest/admin/evacuate.html
  - https://docs.openstack.org/nova/latest/admin/node-down.html
  - https://docs.openstack.org/nova/latest/configuration/config.html#DEFAULT.service_down_time
  - https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/compute-service.html
authored_by: agent:ozarland-author
last_verified: 2026-10-22
---

# nova-compute is down on a compute, but its instances are still running

"Compute down" in Nova means "Nova has heard nothing from the `nova-compute` agent of this host for
`service_down_time` seconds". It does not mean the host is dead. When only the agent stopped, the hypervisor and the
instances run on, and their users notice nothing. The danger is treating it as a dead host: evacuating now would
start a second copy of each instance.

## Symptom

```text
$ openstack compute service list --service nova-compute -c Host -c Status -c State
| compute-1 | enabled | up   |
| compute-2 | enabled | down |
```

The instances of `compute-2` still answer on the network. The host answers to ping and SSH. Every other service is
healthy.

## Impact

- Running instances: none, as long as nobody acts on the wrong diagnosis. Their data plane keeps working.
- Control plane for that host: Nova cannot act on its instances while the agent is down (start, stop, reboot,
  resize, migration, console). New instances are not scheduled there while the service is down.
- Risk: evacuating this host, by hand or by automation that reacts to a lost heartbeat, would rebuild each instance
  on another compute while the original still runs, both writing to the same volumes or ports. That corrupts data.
- Known Nova behaviour, not tested on the lab: deleting an instance whose compute service is down removes it from
  the database only ("local delete"); the domain keeps running on the host until `nova-compute` comes back and
  cleans it up.

## Quick checks

On a controller or the deploy host, with admin credentials:

```bash
# 1. Which compute is down, and since when
openstack compute service list --service nova-compute --long

# 2. What Nova places on that host
openstack server list --all-projects --host <compute-host> -c ID -c Name -c Status
```

From the controller, then on the compute:

```bash
# 3. Is the host alive? It is, in this case.
ping -c 3 <compute-host>
ssh <admin-user>@<compute-host> uptime
```

On the compute:

**Kolla-Ansible:**

```bash
# 4. Is the agent stopped while libvirt runs?
docker ps -a --filter name='^nova_compute$' --filter name='^nova_libvirt$' --format '{{.Names}}\t{{.Status}}'
systemctl is-active kolla-nova_compute-container.service

# 5. Do the instances run in libvirt?
docker exec nova_libvirt virsh list --all
```

**Any:** *(not run on the lab)*

```bash
systemctl is-active nova-compute libvirtd
virsh list --all
```

This page applies when, on the compute that Nova reports down: the host answers, `nova_compute` is not running,
`nova_libvirt` runs, and `virsh list` shows at least one domain `running`, or `paused`. A domain paused on an I/O
error counts like a running one: it keeps its QEMU process and its disk locks, so evacuating it is just as dangerous
(see [compute disk full](../lifecycle/compute-disk-full-instance-paused.md)).

If the host does not answer at all, this is not this page: see
[Compute host failure: fence before you evacuate](compute-host-failure-fence-before-evacuate.md). If it answers on
TCP but refuses your SSH login, it is alive: see
[nova-compute down, host alive, SSH refused](nova-compute-down-host-alive-ssh-refused.md).

## Diagnosis

### Nova agent (on the compute)

Find why the agent stopped, before or right after starting it again.

**Kolla-Ansible:**

```bash
# Exit code and whether the kernel killed it for memory
docker inspect -f '{{.State.Status}} exit={{.State.ExitCode}} oom={{.State.OOMKilled}} finished={{.State.FinishedAt}}' nova_compute

# The unit that manages the container
systemctl status kolla-nova_compute-container.service --no-pager
journalctl -u kolla-nova_compute-container.service --since -2h --no-pager | tail -n 50

# The last lines the agent wrote
tail -n 100 /var/log/kolla/nova/nova-compute.log
grep -E 'ERROR|CRITICAL|AMQP|MessagingTimeout' /var/log/kolla/nova/nova-compute.log | tail -n 20
```

**Any:** *(not run on the lab)*

```bash
systemctl status nova-compute --no-pager
journalctl -u nova-compute --since -2h --no-pager | tail -n 50
tail -n 100 /var/log/nova/nova-compute.log
```

Out-of-memory kills also show in the kernel log of the compute:

```bash
journalctl -k --since -2h --no-pager | grep -i -E 'out of memory|oom-kill|killed process'
```

Exit code 137 means the process was killed with SIGKILL: by the kernel out-of-memory killer (`oom=true`), or by
the container engine when a stop did not finish in time.

### Nova (controller)

```bash
# Last report of the service, and whether someone forced it down
openstack compute service list --service nova-compute --host <compute-host> --long
```

`Forced Down` `True` means an operator or a tool marked the service down on purpose: find out who and why before
going further. That is the step that precedes an evacuation.

## Possible causes

| Cause | How to tell |
|---|---|
| `nova_compute` crashed | Container exited with an error code, traceback at the end of `nova-compute.log` |
| Killed for lack of memory | `OOMKilled` `true`, `oom-kill` lines in the kernel log |
| Stopped and not started again (maintenance, failed restart after an upgrade) | Container exited cleanly or with 137 at a time matching a change; unit `inactive` or `failed` |
| Agent running but cut off from RabbitMQ | Container `Up`, but `nova-compute.log` shows AMQP or `MessagingTimeout` errors and no state report |

In every case the host and its instances are alive. The fix is the agent, never an evacuation.

## Resolution

### Step 1: prove the instances run

Do this first, before any other action on the host: `virsh list --all` on the compute (Quick checks, step 5), or
the console of a sample of instances. If they run, the host is alive. Do not evacuate.

### Step 2: start the agent again

> ⚠️ **State-changing.** Starts `nova-compute` on this compute. Running instances are not touched. On start the
> agent resynchronises with Nova: it reconciles power states, and cleans up the domains of instances that were
> deleted or evacuated while it was down. Undo: stop the unit again (not useful).

On the compute:

**Kolla-Ansible:**

```bash
sudo systemctl start kolla-nova_compute-container.service
```

**Any:** *(not run on the lab)*

```bash
sudo systemctl start nova-compute
```

If the container runs but is cut off from RabbitMQ, restart it instead:

> ⚠️ **State-changing.** Restarts `nova-compute` on this compute. Running instances are not touched; operations in
> progress on this host's instances (a build, a migration) can fail and need to be retried.

**Kolla-Ansible:**

```bash
sudo systemctl restart kolla-nova_compute-container.service
```

**Any:** *(not run on the lab)*

```bash
sudo systemctl restart nova-compute
```

Then fix what stopped it (memory pressure, configuration error after an upgrade, RabbitMQ connectivity). If it
stops again, `nova-compute.log` from the start of the container usually names the reason.

### If the host must be taken out anyway

*(Not run on this page's lab; see the fencing page for its own lab runs.)*

Evacuate only a host that is fenced: powered off or isolated through a channel that does not depend on it (IPMI or
BMC, the hypervisor of a virtual host, the switch port), so that nothing on it can run again. Follow
[Compute host failure: fence before you evacuate](compute-host-failure-fence-before-evacuate.md). Evacuation also
needs shared or recoverable storage for the instances: a local disk is rebuilt from the image and its data is lost.

## Verification

On a controller:

```bash
# The service reports again; expect State up within a minute or so
openstack compute service list --service nova-compute --host <compute-host>

# Instances Nova places on the host, and their power state
openstack server list --all-projects --host <compute-host> --long -c ID -c Status -c "Power State"
```

On the compute:

```bash
# Kolla-Ansible
docker ps --filter name='^nova_compute$' --format '{{.Names}}\t{{.Status}}'
docker exec nova_libvirt virsh list --all
```

Expected: the service `up`, `nova_compute` `Up`, and libvirt running as many domains as Nova shows instances
running on the host: each instance runs once. Compare the counts, and for any doubt the IDs (`virsh domuuid
<domain>` gives the instance ID).

## Lab observations

On the lab, the failure was produced by stopping the `nova_compute` container on compute-2 while an instance ran
there, leaving `nova_libvirt` untouched.

| | Before | During (120 s) | After |
|---|---|---|---|
| `nova_compute` on compute-2 | up | **Exited (137)** | up |
| Compute service of compute-2 | up | **down** | up |
| Compute service of compute-1 | up | up | up |
| Instances in libvirt on compute-2 | 1 running | **1 running** (first build), **2 running** (two later builds) | 1 running |

What `virsh list` showed on compute-2 during the failure:

```text
nova_compute Exited (137)
 Id   Name                State
 1    instance-00000036   running
```

- Nova reported the service down about a minute after the agent stopped, which matches the default
  `service_down_time` of 60 s.
- Starting the agent again through its systemd unit brought the service back `up`; the whole undo, including the
  wait for `up` and the deletion of a test instance, took 34 s on the first build.
- In the first build the test instance failed to build before the agent was stopped (a fault 500 in the compute's
  `_build_resources`, cause not established); the instance running during the failure was another instance already
  on that compute. The situation was observed all the same: Nova reported the compute down while an instance ran
  on it. In the two later builds the test instance built, and two instances ran during the failure.
- In a related run on the same configuration (see
  [nova-compute down, host alive, SSH refused](nova-compute-down-host-alive-ssh-refused.md)), the instance stayed
  `ACTIVE`, power state Running, in Nova while its agent was down.
- Last observed run: 2026-10-13.

## Validity and limits

- Observed on three lab builds with the configuration above: OpenStack 2026.1, Kolla-Ansible 22.2.0, ml2/OVS,
  Docker, Ubuntu 24.04, 3 controllers and 2 computes, local instance disks, no Cinder.
- Also observed in production, on a multinode Kolla-Ansible platform on an earlier release (2024.x or earlier,
  exact release not recorded), by @soufian-zaouam, who confirmed on 2026-10-24 that the page matches; the page's
  steps were not re-run there.
- Not tested: OVN, other releases, other deployment tools (the **Any:** commands are standard but were not run),
  production scale, a host with many instances.
- Observed: the symptom, the signature (service down, container exited with 137, instances running in libvirt) and
  the fix by starting `nova_compute` again.
- Not observed on those builds: the real causes listed under Possible causes (the lab stopped the container on
  purpose), the restart for a RabbitMQ disconnection, a domain paused on an I/O error at the same time, the local
  delete behaviour, and the fencing and evacuation branch.
- A lab review on 2026-10-22 (all-in-one, agent stopped through its unit, see `confirmed_by`) ran the full procedure
  and observed the local delete of an instance while the agent was down; its domain was destroyed when the agent
  came back. Whether it ran the other items above is not recorded.

## Related

- [Compute host failure: fence before you evacuate](compute-host-failure-fence-before-evacuate.md): the case where
  the host is really gone.
- [nova-compute down, host alive, SSH refused](nova-compute-down-host-alive-ssh-refused.md): the same failure on a
  host that refuses your SSH login.
- [Compute disk full, instance paused](../lifecycle/compute-disk-full-instance-paused.md): domains paused on an I/O
  error.
- [Restarting nova-libvirt does not stop instances](../lifecycle/restarting-nova-libvirt-does-not-stop-instances.md)
- [Stale records](../lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md): what a local delete or
  an interrupted operation leaves behind.
- [Control plane down: what still works for instances](../diagnostics/control-plane-down-what-still-works-for-instances.md)
- Nova: [Evacuate instances](https://docs.openstack.org/nova/latest/admin/evacuate.html),
  [Recover from a failed compute node](https://docs.openstack.org/nova/latest/admin/node-down.html).
- Source: lab reproduction by @soufian-zaouam (OpenStack 2026.1, Kolla-Ansible 22.2.0).

---

*Provenance: written by agent:ozarland-author, from a lab reproduction by @soufian-zaouam on 2026.1.
Reviewed in its pull request (see the page history).
Checked by a human operator: not yet.
Behaviour also reported from production by @soufian-zaouam (Kolla-Ansible 2024.x or earlier, steps not re-run).*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=recovery/nova-compute-down-instances-still-running.md&title=Feedback%3A+recovery/nova-compute-down-instances-still-running.md):
your release, your deployment method, and what happened.
