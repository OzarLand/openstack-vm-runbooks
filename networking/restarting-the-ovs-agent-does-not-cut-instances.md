---
title: I need to restart the Open vSwitch agent on a compute. Its instances keep their network
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [neutron, ovs]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers, 2 computes
  network: ml2/ovs (VXLAN)
  storage: local ephemeral disk on the computes, no Cinder
  os: ubuntu-24.04 (noble), docker
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/neutron/latest/contributor/internals/openvswitch_agent.html
  - https://docs.openstack.org/neutron/latest/configuration/openvswitch-agent.html
  - https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html
last_verified: 2026-10-21
---

# I need to restart the Open vSwitch agent on a compute. Its instances keep their network

A planned operation, not a failure: you must restart `neutron-openvswitch-agent` on a compute (a configuration
change, a stuck agent, or a Neutron reconfigure or upgrade, which restarts it on every compute). This page says
what happens to the instances, what to check first and how to verify afterwards.

## Symptom

"I need to restart the Open vSwitch agent on `<compute>`, which runs instances. Will they lose the network?"

On the lab, they did not: no request was lost during the restart.

## Impact

- **Running instances:** no interruption measured. The flows the agent programmed stay in ovs-vswitchd while the
  agent is down.
- **Changes to ports on that compute wait** until the agent is back: new instances (they wait for their port to be
  wired), new or changed security group rules, port updates. Expect this for as long as the restart (about a
  minute on the lab); it was not measured.
- **The restart itself takes about a minute** with Kolla-Ansible (the container's graceful stop timeout,
  `docker_graceful_timeout`, 60 s by default). Allow for it in automations.
- Instances on other computes are not affected.

## Quick checks

Before the restart. On a controller, with admin credentials:

```bash
# 1. The agent of the compute and its state
openstack network agent list --agent-type open-vswitch --host <compute>

# 2. What runs on the compute, and whether anything is being built or migrated right now
openstack server list --all-projects --host <compute> --long -c ID -c Name -c Status -c "Task State"
```

On the compute:

```bash
# 3. ovs-vswitchd runs: this is what keeps the instances connected during the restart
docker exec openvswitch_db ovs-appctl -t ovs-vswitchd version

# 4. drop_flows_on_start is not enabled (no output, or "False", means the default: flows are kept)
docker exec neutron_openvswitch_agent grep -rn drop_flows_on_start /etc/neutron/
```

If ovs-vswitchd does not answer, stop here: you are in
[ovs-vswitchd down on a compute](ovs-vswitchd-down-on-compute.md), and restarting the agent will not help.

## Diagnosis

### Why the instances keep their network

The agent does not carry traffic; ovs-vswitchd does. The agent only programs flows into it. When the agent
restarts, the flows stay. On start, the agent marks the flows it programs with a new cookie, resynchronises with
`neutron-server`, then removes only the flows that still carry the old cookie. Traffic keeps following the old flows
until the new ones are in place. This is the graceful restart of the Open vSwitch agent.

It only holds when `[agent] drop_flows_on_start` is `False` (the default). With `True`, the agent clears all flows
when it starts and the instances lose the network until it has programmed them again.

### Neutron

To follow the restart, on the compute:

```bash
docker logs --tail 50 -f neutron_openvswitch_agent
tail -f /var/log/kolla/neutron/neutron-openvswitch-agent.log
```

## Possible causes

This page has no failure cause. The reasons to restart the agent are:

### Cause 1: planned restart

A configuration change, a Neutron reconfigure or upgrade. Follow the resolution below.

### Cause 2: the agent is stuck or reported dead while ovs-vswitchd runs

The agent is not alive in `openstack network agent list`, but ovs-vswitchd answers (Quick check 3). A restart of the
agent is the usual first step; this case itself was not reproduced on the lab, only the restart of a healthy agent.

## Resolution

### Restart the agent

On the compute.

> ⚠️ **State-changing.** Restarts the Open vSwitch agent of this compute. Running instances were not cut on the lab,
> but port changes on this compute (new instances, security group updates) wait about a minute until the agent is
> back. Do not run it if `drop_flows_on_start` is `True`: the instances would lose the network until the flows are
> programmed again. Undo: none needed; if the agent does not come back, start its unit.

**Kolla-Ansible:**

```bash
systemctl restart kolla-neutron_openvswitch_agent-container.service
```

**Any:**

```bash
systemctl restart neutron-openvswitch-agent
```

On the lab, the restart of the unit took about 70 s, most of it the graceful stop. Whether the agent exited on its
own before Docker killed it at the end of the timeout was not checked.

If the agent does not come back:

> ⚠️ **State-changing.** Starts the Open vSwitch agent of this compute. Affects port wiring on this compute only.
> Undo: stop the same unit.

```bash
systemctl start kolla-neutron_openvswitch_agent-container.service
```

## Verification

On a controller:

```bash
# The agent is alive again (allow about a minute)
openstack network agent list --agent-type open-vswitch --host <compute>

# The instances of the compute answer
ping -c 10 <instance-address>
```

On the compute:

```bash
docker ps --filter name=neutron_openvswitch_agent --format '{{.Names}} {{.Status}}'
docker exec openvswitch_vswitchd ovs-ofctl dump-flows br-int | head    # flows present
```

To measure the interruption yourself, start `ping -i 0.5 -D <instance-address>` from a host that reaches the
instance before the restart and read the sequence numbers afterwards.

## Lab observations

On the lab, the agent's systemd unit was restarted on one compute, as a reconfigure or upgrade does. A request was
sent every 0.5 s to one instance on each compute.

| | Before | During (60 s after the restart) | After (+60 s) |
|---|---|---|---|
| Instance on compute-1 (restarted) | no loss | **295 requests, 0 lost** | 449 requests, 0 lost |
| Instance on compute-2 | no loss | 295 requests, 0 lost | 449 requests, 0 lost |
| Open vSwitch agents | 5 alive | 5 alive | 5 alive |

- No request was lost. A request every 0.5 s cannot see an interruption shorter than that.
- The agents were all reported alive at each observation. Most likely the agent was back before `neutron-server`
  declared it dead (`agent_down_time`); the exact moment it was reported down, if at all, was not recorded.
- The restart of the unit took about 70 s.

## Validity and limits

- Observed on one lab build with the configuration above: OpenStack 2026.1, Kolla-Ansible 22.2.0, ML2/OVS with
  VXLAN, Docker, Ubuntu 24.04, `drop_flows_on_start` at its default.
- Observed: the restart of the agent's unit, no loss on the instances, agents alive afterwards.
- Not run: Quick check 4 (`drop_flows_on_start`), the flow dump during the restart, the restart of a stuck agent
  (cause 2), the `Any:` commands, creating an instance or changing a security group during the restart.
- Not tested: OVN (no Open vSwitch agent there), other releases, other deployment tools, production scale (many
  ports make the resynchronisation longer), DPDK datapaths, `drop_flows_on_start = True`.
- An interruption shorter than 0.5 s could have been missed.

## Related

- [ovs-vswitchd down on a compute](ovs-vswitchd-down-on-compute.md): ovs-vswitchd, unlike the agent, cannot stop
  without cutting the instances.
- [Restarting nova_libvirt does not stop instances](../lifecycle/restarting-nova-libvirt-does-not-stop-instances.md):
  the same question for libvirt.
- [Instance stuck in build waiting for vif-plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md):
  what a new instance does while the agent is away for too long.
- [Moving instances off a host for maintenance](../migration/moving-instances-off-a-host-for-maintenance.md)
- Neutron, Open vSwitch agent internals: https://docs.openstack.org/neutron/latest/contributor/internals/openvswitch_agent.html
- Neutron, `openvswitch_agent.ini` options (`drop_flows_on_start`): https://docs.openstack.org/neutron/latest/configuration/openvswitch-agent.html
- Source: lab reproduction by the OzarLand maintainer (OpenStack 2026.1, Kolla-Ansible 22.2.0).
