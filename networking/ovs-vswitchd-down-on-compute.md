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
  - { who: "lab:ozarland-maintainer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, 3 controllers + 2 computes, OVS, Docker, Ubuntu Noble", date: 2026-10-09, note: "failure injected and observed on 1 lab build; symptom and signature observed (instances of one compute unreachable, its OVS agent not alive, Nova ACTIVE); fix = starting ovs-vswitchd again observed, instances recovered without further action; root-cause search (logs, OOM) not run" }
upstream_refs:
  - https://docs.openstack.org/neutron/latest/admin/deploy-ovs-selfservice.html
  - https://docs.openstack.org/neutron/latest/contributor/internals/openvswitch_agent.html
  - https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html
last_verified: 2026-10-21
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

The loss does not start at once. On the lab, traffic kept flowing for about 25 s after ovs-vswitchd stopped, then
stopped completely.

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
# 1. The Open vSwitch agent of the compute is not alive
openstack network agent list --agent-type open-vswitch --host <compute>

# 2. The instances of that compute are still ACTIVE in Nova
openstack server list --all-projects --host <compute> --long -c ID -c Name -c Status -c "Power State"
```

Run on the compute:

```bash
# 3. The openvswitch_vswitchd container is stopped (openvswitch_db still runs)
docker ps -a --filter name=openvswitch --format '{{.Names}} {{.Status}}'

# 4. ovs-vswitchd does not answer, while ovsdb-server does
docker exec openvswitch_db ovs-appctl -t ovs-vswitchd version   # fails while ovs-vswitchd is down
docker exec openvswitch_db ovs-vsctl show                       # still answers
```

If all four match, you are looking at this problem.

## Diagnosis

### Open vSwitch

On the compute, find why ovs-vswitchd stopped:

```bash
# State and recent history of the service
systemctl status kolla-openvswitch_vswitchd-container.service
journalctl -u kolla-openvswitch_vswitchd-container.service --since "-1h"

# Log of ovs-vswitchd itself
docker logs --tail 200 openvswitch_vswitchd
tail -n 200 /var/log/kolla/openvswitch/ovs-vswitchd.log

# Killed by the kernel for lack of memory?
dmesg -T | grep -i -E 'out of memory|killed process'
```

**Any:** with a packaged Open vSwitch, the service is `ovs-vswitchd` (or `openvswitch-switch` on Ubuntu); read
`journalctl -u ovs-vswitchd` and `/var/log/openvswitch/ovs-vswitchd.log`.

### Neutron

The Open vSwitch agent cannot program flows without ovs-vswitchd, so it is reported dead. Its log on the compute
(`/var/log/kolla/neutron/neutron-openvswitch-agent.log`) is expected to show errors talking to Open vSwitch (this log
was not read on the lab). The dead agent is a consequence, not the cause.

### Nova

Nothing. Nova reads the state of the instances from libvirt, and libvirt still runs them.

## Possible causes

### Cause 1: ovs-vswitchd stopped or was killed

The `openvswitch_vswitchd` container is stopped or restarting. The reason is in its log, the journal of its unit, or
`dmesg` (an out-of-memory kill). This is the case observed on the lab.

### Cause 2: ovs-vswitchd runs but is stuck

The container is up but `ovs-appctl -t ovs-vswitchd version` hangs or times out. Not observed on the lab; the
resolution below (a restart instead of a start) is expected to apply, but was not tested.

## Resolution

### For cause 1

On the compute, start ovs-vswitchd through its unit.

> ⚠️ **State-changing.** Starts ovs-vswitchd on this compute. Affects the instances of this compute only: their
> traffic resumes once the Open vSwitch agent has programmed the flows again. Undo: none needed; to stop it again,
> stop the same unit (which cuts the instances of the compute again).

**Kolla-Ansible:**

```bash
systemctl start kolla-openvswitch_vswitchd-container.service
```

**Any:**

```bash
systemctl start ovs-vswitchd
```

Then nothing else is needed: the Open vSwitch agent notices the restart of ovs-vswitchd and programs the flows
again. On the lab, the instances answered again about 30 s after the start, without restarting the agent or the
instances.

Before closing the incident, find why it stopped (see Diagnosis). If it was an out-of-memory kill, it can happen
again.

### For cause 2

> ⚠️ **State-changing.** Restarts ovs-vswitchd on this compute. The instances of this compute, already cut, stay cut
> until the agent has programmed the flows again. Undo: none needed.

**Kolla-Ansible:**

```bash
systemctl restart kolla-openvswitch_vswitchd-container.service
```

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
docker exec openvswitch_db ovs-appctl -t ovs-vswitchd version   # answers
docker exec openvswitch_vswitchd ovs-ofctl dump-flows br-int | head
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

## Validity and limits

- Observed on one lab build with the configuration above: OpenStack 2026.1, Kolla-Ansible 22.2.0, ML2/OVS with
  VXLAN, Docker, Ubuntu 24.04. Host addresses on plain interfaces, not on an OVS bridge.
- Observed: the symptom, the signature (agent not alive, container stopped, Nova unaware), the recovery by starting
  the unit again.
- Not run: the root-cause search in logs and `dmesg` (the lab stopped the unit on purpose), cause 2 (stuck
  ovs-vswitchd), the `Any:` commands.
- Not tested: OVN, other releases, other deployment tools, production scale, a host whose own address sits on an OVS
  bridge (the host would lose its connectivity too, and you would need console access), DPDK datapaths.
- The 25 s delay before the cut and the 30 s recovery are lab figures; they depend on traffic and on the number of
  ports.

## Related

- [Restarting the Open vSwitch agent does not cut instances](restarting-the-ovs-agent-does-not-cut-instances.md):
  the agent, unlike ovs-vswitchd, can be restarted safely.
- [New instance gets no IP: DHCP agents down](new-instance-gets-no-ip-dhcp-agents-down.md)
- [Instance stuck in build waiting for vif-plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md):
  what new instances on this compute do while the agent is dead.
- [nova-compute down, instances still running](../recovery/nova-compute-down-instances-still-running.md): the
  opposite case, control plane of a compute down, data plane fine.
- Neutron, Open vSwitch self-service networks: https://docs.openstack.org/neutron/latest/admin/deploy-ovs-selfservice.html
- Neutron, Open vSwitch agent internals: https://docs.openstack.org/neutron/latest/contributor/internals/openvswitch_agent.html
- Source: lab reproduction by the OzarLand maintainer (OpenStack 2026.1, Kolla-Ansible 22.2.0).
