---
title: Every instance on one compute loses the network while Nova shows them ACTIVE (ovs-vswitchd down)
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [neutron, ovs]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers, 2 computes
  network: ml2/ovs (VXLAN)
  storage: local ephemeral disk on the computes, no Cinder
  os: ubuntu-24.04 (noble), docker
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, multinode (3 controllers, 2 computes), ML2/OVS VXLAN", date: 2026-10-27, note: "full procedure on a disposable lab for cause 1 (unit stopped, then started; Diagnosis commands with sudo, agent log read); cause 2 simulated by freezing ovs-vswitchd with kill -STOP, resolved by the restart; the Any: commands, a real OOM kill or hang, and OVN not run" }
upstream_refs:
  - https://docs.openstack.org/neutron/latest/admin/deploy-ovs-selfservice.html
  - https://docs.openstack.org/neutron/latest/contributor/internals/openvswitch_agent.html
  - https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Every instance on one compute loses the network while Nova shows them ACTIVE (ovs-vswitchd down)

ovs-vswitchd is the process that switches the traffic of the instances on a compute (`br-int`, `br-tun`). When it
stops, every instance of that compute loses the network, but Nova sees nothing.

## Symptom

- Every instance of one compute stops answering on the network (ping, SSH, application traffic).
- Instances on other computes are fine.
- In Nova the affected instances are still `ACTIVE`, power state `Running`.
- The Open vSwitch agent of that compute is reported not alive:

```text
$ openstack network agent list --agent-type open-vswitch
+------+--------------------+-----------+-------+-------+---------------------------+
| ID   | Agent Type         | Host      | Alive | State | Binary                    |
+------+--------------------+-----------+-------+-------+---------------------------+
| ...  | Open vSwitch agent | compute-1 | XXX   | UP    | neutron-openvswitch-agent |
| ...  | Open vSwitch agent | compute-2 | :-)   | UP    | neutron-openvswitch-agent |
...
```

The loss does not start at once. On two lab runs, traffic kept flowing for about 25 s and for 6 s after ovs-vswitchd
stopped, then stopped completely. The agent is not reported dead at once either: on the second run it still showed
`:-)` 53 s after the stop and `XXX` 65 s after.

## Impact

- Data plane down for **every instance of that compute**. Nothing is lost on disk; the instances keep running.
- The host itself keeps its connectivity when its management and tunnel addresses are on plain interfaces, not on
  an Open vSwitch bridge (as on the lab). SSH to the compute still works.
- Port changes on that compute (new instances, security group changes) wait, because the Open vSwitch agent depends
  on ovs-vswitchd.
- Monitoring based on Nova states does not see it.

## Quick checks

Run on a controller, with admin credentials:

```bash
# 1. The Open vSwitch agent of the compute is not alive (can take about a minute after the stop)
openstack network agent list --agent-type open-vswitch --host <compute>

# 2. The instances of that compute are still ACTIVE in Nova
openstack server list --all-projects --host <compute> --long -c ID -c Name -c Status -c "Power State"
```

Run on the compute:

```bash
# 3. The openvswitch_vswitchd container is stopped (openvswitch_db still runs)
sudo docker ps -a --filter name=openvswitch --format '{{.Names}} {{.Status}}'

# 4. ovs-vswitchd does not answer, while ovsdb-server does
sudo docker exec openvswitch_vswitchd ovs-appctl version   # fails while ovs-vswitchd is down (container not running)
sudo docker exec openvswitch_db ovs-vsctl show             # still answers
```

On the lab, `docker` without `sudo` is denied (`permission denied while trying to connect to the docker API`). The
filter also lists `neutron_openvswitch_agent`. With the unit stopped, check 3 shows `openvswitch_vswitchd Exited (0)`
and check 4 prints `Error response from daemon: container <container-id> is not running`.

`ovs-appctl` runs in the `openvswitch_vswitchd` container. From `openvswitch_db`, `ovs-appctl -t ovs-vswitchd` reads a
pidfile that is not valid there and fails even while ovs-vswitchd runs (`stale pidfile`, `cannot read pidfile`),
as a lab review on Kolla-Ansible 22.2.0 found.

If all four match, you are looking at this problem.

## Diagnosis

### Open vSwitch

On the compute, find why ovs-vswitchd stopped:

```bash
# State and recent history of the service
systemctl status kolla-openvswitch_vswitchd-container.service
sudo journalctl -u kolla-openvswitch_vswitchd-container.service --since "-1h"

# Log of ovs-vswitchd itself
sudo docker logs --tail 200 openvswitch_vswitchd
sudo tail -n 200 /var/log/kolla/openvswitch/ovs-vswitchd.log

# Killed by the kernel for lack of memory?
sudo dmesg -T | grep -i -E 'out of memory|killed process'
```

Without `sudo`, `tail` gives `Permission denied` and `dmesg` gives `read kernel buffer failed: Operation not
permitted`. After a plain stop of the unit (as on the lab), the journal shows `Stopping`, `Deactivated successfully`,
`Stopped`, the container `Exited (0)`, and `dmesg` has no out-of-memory line.

**Any:** with a packaged Open vSwitch, the service is `ovs-vswitchd` (or `openvswitch-switch` on Ubuntu); read
`journalctl -u ovs-vswitchd` and `/var/log/openvswitch/ovs-vswitchd.log`.

### Neutron

The Open vSwitch agent cannot program flows without ovs-vswitchd, so it is reported dead. Its log on the compute
(`sudo grep -E 'OVS is|Failed to communicate' /var/log/kolla/neutron/neutron-openvswitch-agent.log`) shows, as
captured on the lab:

```text
ERROR ...openflow.native.br_int [...] Failed to communicate with the switch: RuntimeError: ofctl request ...
WARNING ...ovs_neutron_agent [...] OVS is dead. OVSNeutronAgent will keep running and checking OVS status periodically.
```

and once ovs-vswitchd is back, `OVS is restarted. OVSNeutronAgent will reset bridges and recover ports.` then
`Agent has just been revived. Doing a full sync.` The dead agent is a consequence, not the cause.

### Nova

Nothing. Nova reads the state of the instances from libvirt, and libvirt still runs them.

## Possible causes

### Cause 1: ovs-vswitchd stopped or was killed

The `openvswitch_vswitchd` container is stopped or restarting. The reason is in its log, the journal of its unit, or
`dmesg` (an out-of-memory kill). This is the case observed on the lab.

### Cause 2: ovs-vswitchd runs but is stuck

The container is up but `sudo docker exec openvswitch_vswitchd ovs-appctl version` hangs or times out (wrap it in
`timeout 10`). The container still reports `Up (healthy)`: on the lab, with ovs-vswitchd frozen by `kill -STOP`, it
stayed healthy for the 52 s observed, `ovs-appctl version` gave no answer in 10 s, `ovs-vsctl show` in
`openvswitch_db` still answered, and the agent logged nothing until the restart. Simulated on the lab only (a frozen
process, not a real hang).

## Resolution

### For cause 1

On the compute, start ovs-vswitchd through its unit.

> ⚠️ **State-changing.** Starts ovs-vswitchd on this compute. Affects the instances of this compute only: their
> traffic resumes once the Open vSwitch agent has programmed the flows again. Undo: none needed; to stop it again,
> stop the same unit (which cuts the instances of the compute again).

**Kolla-Ansible:**

```bash
sudo systemctl start kolla-openvswitch_vswitchd-container.service
```

**Any:**

```bash
systemctl start ovs-vswitchd
```

Then nothing else is needed: the Open vSwitch agent notices the restart of ovs-vswitchd and programs the flows
again. On the lab, the instances answered again about 30 s after the start on one run and 3 s after it on another,
without restarting the agent or the instances. Without `sudo`, `systemctl start` fails with `Interactive
authentication required.`

Before closing the incident, find why it stopped (see Diagnosis). If it was an out-of-memory kill, it can happen
again.

### For cause 2

> ⚠️ **State-changing.** Restarts ovs-vswitchd on this compute. The instances of this compute, already cut, stay cut
> until the agent has programmed the flows again. Undo: none needed.

**Kolla-Ansible:**

```bash
sudo systemctl restart kolla-openvswitch_vswitchd-container.service
```

On the lab (frozen ovs-vswitchd), the restart returned in about 1 s and the instance answered again 2.6 s later. For
a few seconds after the restart, `ovs-appctl version` prints `cannot read pidfile
"/var/run/openvswitch/ovs-vswitchd.pid" (No such process)`; it answers once ovs-vswitchd has started.

**Any:**

```bash
systemctl restart ovs-vswitchd
```

## Verification

On a controller:

```bash
# The agent is alive again
openstack network agent list --agent-type open-vswitch --host <compute>

# The instances answer (from any host that reaches them, or from the router namespace)
ping -c 10 <instance-address>
```

On the compute:

```bash
sudo docker exec openvswitch_vswitchd ovs-appctl version   # answers
sudo docker exec openvswitch_vswitchd ovs-ofctl dump-flows br-int | head
```

Expected: agent `:-)` (alive), pings answered, flows present on `br-int`.

## Lab observations

On the lab, the failure was produced by stopping the systemd unit of the `openvswitch_vswitchd` container on one
compute. A request was sent every 0.5 s to one instance on each compute.

| | Before | During (90 s) | After (start + 60 s) |
|---|---|---|---|
| Instance on compute-1 | answering | **down: 144 of 235 requests lost, gap starting about 25 s after the stop** | answering; one gap of **88.5 s** in all |
| Instance on compute-2 | answering | 235 requests, 0 lost | 396 requests, 0 lost |
| Open vSwitch agents | 5 alive | **compute-1 not alive** | 5 alive |
| Instance on compute-1 in Nova | ACTIVE, running | **ACTIVE, running** | ACTIVE |

- The first 25 s without loss are most likely flows already cached in the kernel datapath. Any packet that needs
  ovs-vswitchd (an upcall, such as an ARP refresh) is dropped, so the cut becomes complete.
- The instances recovered about 30 s after ovs-vswitchd started, with no other action.
- Nova never noticed anything.

A second run on another build of the same lab (2026-10-27), same method, one test instance on each compute, a
request every 0.5 s from the router namespace:

| | Cause 1 (unit stopped) | Cause 2 simulated (`kill -STOP`) |
|---|---|---|
| Loss starts after the failure | 6 s | 17 s |
| Agent of the compute | `:-)` until at least 53 s, not alive at 65 s; alive 6 s after the start | not polled |
| Container status | `Exited (0)` | `Up (healthy)` |
| Instance answers again | 3 s after `systemctl start` (gap 81.4 s) | 2.6 s after `systemctl restart` returned (gap 39.7 s) |
| Instance on the other compute | 0 lost | 0 lost |

## Validity and limits

- Observed on one lab build with the configuration above: OpenStack 2026.1, Kolla-Ansible 22.2.0, ML2/OVS with
  VXLAN, Docker, Ubuntu 24.04. Host addresses on plain interfaces, not on an OVS bridge.
- Observed: the symptom, the signature (agent not alive, container stopped, Nova unaware), the recovery by starting
  the unit again.
- Run on the second lab run: the root-cause commands of Diagnosis (logs, journal, `dmesg`; nothing to find, since
  the unit was stopped on purpose), the Neutron agent log, cause 2 as a simulation (`kill -STOP` on ovs-vswitchd)
  and its restart.
- Not run: the `Any:` commands; a real out-of-memory kill or a real hang of ovs-vswitchd.
- Not tested: OVN, other releases, other deployment tools, production scale, a host whose own address sits on an OVS
  bridge (the host would lose its connectivity too, and you would need console access), DPDK datapaths.
- The delays before the cut (25 s, 6 s, 17 s) and the recovery times (30 s, 3 s) are lab figures; they depend on
  traffic and on the number of ports.

## Related

- [Restarting the Open vSwitch agent does not cut instances](restarting-the-ovs-agent-does-not-cut-instances.md):
  the agent, unlike ovs-vswitchd, can be restarted safely.
- [New instance gets no IP: DHCP agents down](new-instance-gets-no-ip-dhcp-agents-down.md)
- [Instance stuck in build waiting for vif-plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md):
  what new instances on this compute do while the agent is dead.
- [nova-compute down, instances still running](../recovery/nova-compute-down-instances-still-running.md): the
  opposite case, control plane of a compute down, data plane fine.
- [After a compute reboot, instances stay SHUTOFF or come back without network](../recovery/after-a-compute-reboot-instances-shutoff-or-without-network.md):
  when the switch, an interface or the MTU did not come back after a reboot.
- Neutron, Open vSwitch self-service networks: https://docs.openstack.org/neutron/latest/admin/deploy-ovs-selfservice.html
- Neutron, Open vSwitch agent internals: https://docs.openstack.org/neutron/latest/contributor/internals/openvswitch_agent.html
- Source: lab reproduction by @soufian-zaouam (OpenStack 2026.1, Kolla-Ansible 22.2.0).

---

*Provenance: written by agent:ozarland-author, from a lab reproduction by @soufian-zaouam on 2026.1.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=networking/ovs-vswitchd-down-on-compute.md&title=Feedback%3A+networking/ovs-vswitchd-down-on-compute.md):
your release, your deployment method, and what happened.
