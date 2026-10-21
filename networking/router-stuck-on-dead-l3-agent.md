---
title: L3 agent dead, instances still reachable, router not moved
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [neutron]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers, 2 computes
  network: ml2/ovs (VXLAN), legacy routers (no L3 HA, no DVR)
  storage: none (no Cinder on the lab)
  os: ubuntu-24.04 (noble), docker
risk: destructive
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/neutron/latest/admin/deploy-ovs-ha-vrrp.html
  - https://docs.openstack.org/neutron/latest/configuration/neutron.html
  - https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html
last_verified: 2026-10-21
---

# L3 agent dead, instances still reachable, router not moved

The router's namespace, NAT rules and floating IPs are kernel state that the L3 agent set up. They stay when the agent
stops, so traffic keeps flowing. What stops is every change: the router is frozen on a dead agent, and nothing moves
it elsewhere.

## Symptom

- An L3 agent is reported not alive:

```text
$ openstack network agent list --agent-type l3 -c Host -c Alive -c State
| controller-1 | XXX  | UP |
| controller-2 | :-)  | UP |
| controller-3 | :-)  | UP |
```

- Instances behind its routers still answer on their floating IPs.
- Users may report that a new floating IP, a new router interface or a gateway change does not work.

## Impact

Running traffic through the routers of the dead agent continues. Changes to those routers no longer apply. The
platform looks healthy from the data plane, so the failure is silent until someone changes a router. No data is at
risk.

## Quick checks

On a controller, with admin credentials, unless stated otherwise.

```bash
# 1. Which L3 agent is dead
openstack network agent list --agent-type l3

# 2. Where the router lives, and whether that agent is alive (--long requires --router)
openstack network agent list --router <router> --long

# 3. On the agent's host: the namespace is still there
ip netns list | grep qrouter-<router-id>

# 4. On the agent's host: the agent container
docker ps -a --filter name=neutron_l3_agent --format '{{.Names}} {{.Status}}'
```

If the router is listed on the dead agent only, and its namespace is still on that host, this page applies.

## Diagnosis

### Neutron

List every router of the dead agent, the ones that will need attention:

```bash
openstack router list --agent <dead-agent-id>
```

**Kolla-Ansible:** on the agent's host, `systemctl status kolla-neutron_l3_agent-container.service` and
`tail -n 100 /var/log/kolla/neutron/neutron-l3-agent.log` say why it stopped. An exit code of 143 means it was
stopped by a signal (SIGTERM).
**Any:** `journalctl -u neutron-l3-agent`, or the configured log file.

### Is the host alive?

Before moving anything, know whether the host itself is up. The namespace and its floating IPs stay active on a host
that is up, even with its agent dead:

```bash
ping -c 3 <agent-host>
ssh <agent-host> ip netns list
```

### Why Neutron does not move the router

With Kolla-Ansible's default `enable_neutron_agent_ha: false`, a router lives on one L3 agent. Neutron's
`allow_automatic_l3agent_failover` is false by default, so nothing reschedules it when the agent dies.

## Possible causes

### Cause 1: the L3 agent stopped, the host is alive

The container is stopped or the process died; the host answers, and the `qrouter-` namespace is still there and still
carries the traffic. This is the case the lab reproduced.

### Cause 2: the host is lost

The host does not answer at all. The routers it held are frozen and, if the host is really down, their traffic is
interrupted too. Not reproduced on the lab.

## Resolution

### For cause 1: start the agent again

> ⚠️ **State-changing.** Starts the L3 agent; it resyncs the routers it holds. On the lab the instances behind the
> router lost no traffic while it did. Undo: stop the unit again (this reproduces the failure).

**Kolla-Ansible:** on the agent's host:

```bash
systemctl start kolla-neutron_l3_agent-container.service
```

**Any:** `systemctl start neutron-l3-agent` on the agent's host.

Then find why it stopped, from the log and the unit status.

### For cause 2: move the routers to a live agent

> 🛑 **Destructive.** Rebuilds the router on another host: every instance behind it loses its traffic while the router
> is rebuilt, and existing connections through it are cut. Precondition: the old host is really down, or its
> `qrouter-<router-id>` namespace is gone; otherwise the same gateway and floating IPs answer from two hosts at once.
> Tell the owners of the instances behind the router first. Moving the router back later causes a second
> interruption.

```bash
# On a controller, admin credentials
openstack network agent remove router --l3 <dead-agent-id> <router>
openstack network agent add router --l3 <live-agent-id> <router>
openstack network agent list --router <router> --long
```

Repeat for each router of `openstack router list --agent <dead-agent-id>`.

### For the long term

Consider L3 HA (`enable_neutron_agent_ha: true` in Kolla-Ansible), which keeps a standby of each router on another
agent. It changes how routers are built, and existing routers are not converted by the setting alone. Not tested on
the lab: read the upstream documentation first.

## Verification

```bash
# On a controller: the agent alive, and the router on a live agent
openstack network agent list --agent-type l3
openstack network agent list --router <router> --long
```

Then apply a small change to the router, such as a new floating IP on a test instance behind it, and check it answers.
Delete the test floating IP afterwards.

## Lab observations

The failure was produced by stopping the L3 agent unit of the host that held the router of the lab's test instances
(controller-1). A continuous ping to the instances behind that router measured their traffic.

| | Before | During (120 s) | After (agent started + 60 s) |
|---|---|---|---|
| Instances (continuous ping) | no loss | **264 requests each, 0 lost** | 432 requests, 0 lost |
| L3 agents | 3 alive | **controller-1 not alive** | 3 alive |
| Agent of the router | controller-1, alive | **controller-1, not alive** (not moved) | controller-1, alive |
| `neutron_l3_agent` on controller-1 | up | **Exited (143)** | up |

The router stayed on the dead agent for the whole outage, and the instances behind it lost nothing, during the
outage or when the agent started again.

## Validity and limits

- Observed on one lab build with the configuration above: OpenStack 2026.1, Kolla-Ansible 22.2.0, ML2/OVS with VXLAN,
  legacy (non-HA, non-DVR) routers, Docker, Ubuntu Noble, 3 controllers and 2 computes.
- Observed: the symptom, the router left on the dead agent, no traffic lost, and the fix by starting the agent again.
- Not run: a change to the router while the agent was down (the claim that changes do not apply follows from how
  the agent works, it was not observed), moving a router to another agent, L3 HA, the loss of a whole host.
- Not tested: OVN (no L3 agent), DVR, L3 HA routers, `allow_automatic_l3agent_failover`, other releases, other
  deployment tools, production scale.

## Related

- [New instance is ACTIVE but never got an IP address](new-instance-gets-no-ip-dhcp-agents-down.md)
- [Restarting the OVS agent does not cut instances](restarting-the-ovs-agent-does-not-cut-instances.md)
- [Control plane down: what still works for running instances](../diagnostics/control-plane-down-what-still-works-for-instances.md)
- [Compute host failure: fence before you evacuate](../recovery/compute-host-failure-fence-before-evacuate.md): the same
  "is the host really dead?" question, for instances.
- Neutron: [L3 HA with VRRP](https://docs.openstack.org/neutron/latest/admin/deploy-ovs-ha-vrrp.html),
  [configuration reference](https://docs.openstack.org/neutron/latest/configuration/neutron.html)
  (`allow_automatic_l3agent_failover`)
- Source: lab reproduction by the OzarLand maintainer (OpenStack 2026.1, Kolla-Ansible 22.2.0).
