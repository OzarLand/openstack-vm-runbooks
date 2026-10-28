---
title: New instance is ACTIVE but unreachable, it never got an IP address
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [neutron, nova]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers, 2 computes
  network: ml2/ovs (VXLAN)
  storage: none (no Cinder on the lab)
  os: ubuntu-24.04 (noble), docker
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-22, note: "full procedure for cause 1 on a disposable lab, including the reboot of an instance created during the outage; causes 2 to 4 not reproduced" }
upstream_refs:
  - https://docs.openstack.org/neutron/latest/admin/config-dhcp-ha.html
  - https://docs.openstack.org/neutron/latest/configuration/dhcp-agent.html
  - https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html
authored_by: agent:ozarland-author
last_verified: 2026-10-22
---

# New instance is ACTIVE but unreachable, it never got an IP address

Nova did its job: the instance is `ACTIVE`. Its first act, asking for an address by DHCP, gets no answer, because no
DHCP agent is alive to add its port to dnsmasq. Instances created earlier keep their lease and notice nothing.

## Symptom

- A new instance is `ACTIVE`, power state running, no fault, but it does not answer on its fixed or floating IP, and
  SSH fails.
- Instances created earlier work normally.
- The console log of the new instance shows the DHCP client asking and never getting an answer:

```text
$ openstack console log show <server-id> | grep -iE 'dhcp|lease|bound'
...
soliciting a DHCP lease
...
```

There is no line saying a lease was obtained (`leased`, `bound`). The exact wording depends on the image; the line
above is the one CirrOS printed on the lab.

## Impact

Every instance created while the agents are down boots without an address and is unreachable. Running instances are
not affected as long as their lease is valid (`dhcp_lease_duration`, 86400 s by default; they start renewing at
half of it). An outage longer than that would reach running instances too (not tested). No data is at risk.

## Quick checks

On a controller, with admin credentials, unless stated otherwise.

```bash
# 1. The instance is ACTIVE and has no fault
openstack server show <server-id> -c status -c OS-EXT-STS:power_state -c fault -c addresses

# 2. Its console: DHCP asked, never answered
openstack console log show <server-id> | grep -iE 'dhcp|lease|bound'

# 3. The DHCP agents: are any alive?
openstack network agent list --agent-type dhcp

# 4. On each network node: the agent container
docker ps -a --filter name=neutron_dhcp_agent --format '{{.Names}} {{.Status}}'
```

If every DHCP agent is not alive (`Alive` shows `XXX` or `False`), this page applies.

## Diagnosis

### Neutron

Which agents host the network of the instance, and whether they are alive:

```bash
openstack network agent list --network <network>
openstack port list --network <network> --device-owner network:dhcp -c ID -c "Fixed IP Addresses" -c Status
```

On the network node that hosts the network, the namespace and the agent log:

```bash
ip netns list | grep qdhcp-<network-id>
```

**Kolla-Ansible:** `tail -n 100 /var/log/kolla/neutron/neutron-dhcp-agent.log`, and
`systemctl status kolla-neutron_dhcp_agent-container.service` for why the container stopped.
**Any:** `journalctl -u neutron-dhcp-agent` or the configured log file.

The `qdhcp-` namespace and its dnsmasq can stay in place while the agent is down. That is not enough: dnsmasq answers
only the ports the agent wrote to its hosts file, and a new port is never added while the agent is down.

### Nova

Nothing to find: Nova sees a running instance. The instance's console is the only place the failure shows.

## Possible causes

### Cause 1: every DHCP agent is down

`openstack network agent list --agent-type dhcp` shows no agent alive. Every new instance, on every network served by
DHCP, is affected. This is the case the lab reproduced.

### Cause 2: the network has no live DHCP agent, but others are alive

Some DHCP agents are alive, but `openstack network agent list --network <network>` lists only dead ones, or none.
Only new instances on that network are affected. Not reproduced on the lab.

### Cause 3: the agent is alive, but dnsmasq of the network is not running

The agent of the network is alive, but `ps` on the network node shows no dnsmasq for `<network-id>`, or the agent log
shows errors for that network. With Kolla-Ansible's `neutron_agents_wrappers`, dnsmasq runs in a container of its own,
started by the agent. Not reproduced on the lab.

### Cause 4: the subnet does not use DHCP

`openstack subnet show <subnet> -c enable_dhcp` shows `False`. The image must then be configured another way. Not a
fault of the agents.

## Resolution

### For causes 1 and 2: start the DHCP agents, then find why they stopped

*(Run on the lab for cause 1; cause 2 not reproduced.)*

> ⚠️ **State-changing.** Starts the DHCP agent on the network node. When all agents were down, Neutron may schedule
> the networks again on other agents when they come back: their `qdhcp-` namespaces can move to another host (observed
> on the lab). Running instances lost no traffic on the lab while it happened. Undo: stop the unit again (this
> reproduces the outage).

**Kolla-Ansible:** on each network node:

```bash
systemctl start kolla-neutron_dhcp_agent-container.service
```

**Any:** `systemctl start neutron-dhcp-agent` on each network node *(not run on the lab)*.

Then read the agent log and the unit status from before the restart to find why it stopped.

### For the instances created during the outage

Their DHCP client may retry by itself after a while, depending on the image. Otherwise reboot them. On the lab,
CirrOS had not retried 90 s after the agent came back, and got its lease at the next boot.

> ⚠️ **State-changing.** Reboots the instance. Its owner loses whatever runs in it for the time of the boot; nothing
> is lost on disk. Tell the owner first. It cannot be undone, but it changes nothing beyond a normal reboot.

```bash
openstack server reboot <server-id>
```

### For cause 3

*(Not run on the lab.)* Read the agent log for that network. Restarting the agent, as above, makes it start dnsmasq for its networks again.

### For cause 4

Enable DHCP on the subnet, or configure the instance address another way (config drive, image). Out of the scope of
this page.

## Verification

```bash
# On a controller: the agents alive, and the network on a live agent
openstack network agent list --agent-type dhcp
openstack network agent list --network <network>

# A new test instance gets its lease and answers
openstack server create --flavor <flavor> --image <image> --network <network> dhcp-test
openstack console log show dhcp-test | grep -iE 'lease|bound'
openstack server delete dhcp-test
```

The console must show a lease obtained, and the instance must answer on its address.

## Lab observations

The failure was produced by stopping the DHCP agent units on the three network nodes of the lab (the controllers),
then requesting a test instance on compute-1. A continuous ping to the running instances measured their traffic.

| | Before | During (180 s) | After (agents started + 60 s) |
|---|---|---|---|
| Running instances (continuous ping) | no loss | **405 requests each, 0 lost** | **588 requests each, 0 lost**, the restart of the agents included |
| DHCP agents | 3 alive | **3 not alive** | 3 alive |
| Test instance | none | **ACTIVE in 15 s, no address**; console: `soliciting a DHCP lease`, no lease in 180 s | deleted |

- A first run on another lab build gave the same result: test instance `ACTIVE` in 13 s with no address, running
  instances 412 requests each, 0 lost, while the agents were down.
- When the agents were started again, both networks of the lab moved to another DHCP agent: their `qdhcp-`
  namespaces were on controller-2 before and on controller-3 after. Kolla-Ansible sets `dhcp_agents_per_network`
  only with `enable_neutron_agent_ha`; otherwise Neutron's default of one agent per network applies.
- On that first run the traffic measurement pinged from inside the `qdhcp-` namespace of controller-2, and lost it
  when the network moved. It was a flaw of the measurement, not of the instances; the second run, from a namespace of
  its own, confirmed the instances lost nothing.

A lab review on a second platform (all-in-one, one DHCP agent, same releases) repeated cause 1 for about 7.5 minutes:

- running instances lost nothing (1328 requests, 0 lost), the agent was alive 23 s after its unit was started;
- the dnsmasq and haproxy wrapper containers (`neutron_dhcp_agent_dnsmasq_qdhcp-<network-id>`,
  `neutron_dhcp_agent_haproxy_qdhcp-<network-id>`) stayed `Up` while the agent was down: dnsmasq keeps running but
  never learns the new port;
- an instance created during the outage still had no lease 90 s after the restart (CirrOS's DHCP client had timed
  out), and got it after `openstack server reboot`;
- with a single agent, the network stayed on the same agent: the move between agents above needs several agents.

## Validity and limits

- Observed on lab builds with the configuration above: OpenStack 2026.1, Kolla-Ansible 22.2.0, ML2/OVS with VXLAN,
  Docker, Ubuntu Noble, 3 controllers and 2 computes, CirrOS test instances, default `neutron_agents_wrappers`.
- Observed: the symptom and signature, running instances unaffected during the outage and the restart, the fix by
  starting the agents, and the networks moving to another DHCP agent on restart.
- Also observed, on an all-in-one lab: the reboot of an instance created during the outage (CirrOS), a new instance
  getting its lease after the restart, and the wrapper containers staying up. Causes 2, 3 and 4 were not reproduced.
- The networks moving to another DHCP agent rests on the multinode runs only.
- Not tested: an outage longer than the lease, DHCP agent HA (`enable_neutron_agent_ha`), OVN (which has no DHCP
  agent), cloud-init images, other releases, other deployment tools, production scale.

## Related

- [New instance boots without its metadata](new-instance-boots-without-metadata.md): the same "ACTIVE but broken"
  pattern, with the metadata agents.
- [Router stuck on a dead L3 agent](router-stuck-on-dead-l3-agent.md)
- [Control plane down: what still works for running instances](../diagnostics/control-plane-down-what-still-works-for-instances.md)
- [What a VM creation traverses, and tracing it by request id](../diagnostics/what-a-vm-creation-traverses-and-tracing-by-request-id.md)
- Neutron: [DHCP agent high availability](https://docs.openstack.org/neutron/latest/admin/config-dhcp-ha.html),
  [DHCP agent configuration](https://docs.openstack.org/neutron/latest/configuration/dhcp-agent.html)
- Source: lab reproduction by @soufian-zaouam (OpenStack 2026.1, Kolla-Ansible 22.2.0).

---

*Provenance: written by agent:ozarland-author, from a lab reproduction by @soufian-zaouam on 2026.1.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=networking/new-instance-gets-no-ip-dhcp-agents-down.md&title=Feedback%3A+networking/new-instance-gets-no-ip-dhcp-agents-down.md):
your release, your deployment method, and what happened.
