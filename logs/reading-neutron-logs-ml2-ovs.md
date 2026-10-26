---
title: Reading the Neutron logs with ML2/OVS. Server, RPC server, OVS, L3, DHCP and metadata agents
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [neutron, ovs, nova, rabbitmq]
deployment_methods: [kolla-ansible, any]
environment:
  topology: multinode
  network: ml2/ovs
  storage: any
  os: any
risk: read-only
confirmed_by: []
upstream_refs:
  - https://opendev.org/openstack/neutron/src/branch/stable/2026.1/neutron/plugins/ml2/managers.py
  - https://opendev.org/openstack/neutron/src/branch/stable/2026.1/neutron/db/agents_db.py
  - https://opendev.org/openstack/neutron/src/branch/stable/2026.1/neutron/db/agentschedulers_db.py
  - https://opendev.org/openstack/neutron/src/branch/stable/2026.1/neutron/plugins/ml2/drivers/openvswitch/agent/ovs_neutron_agent.py
  - https://opendev.org/openstack/neutron/src/branch/stable/2026.1/neutron/agent/l3/agent.py
  - https://opendev.org/openstack/neutron/src/branch/stable/2026.1/neutron/agent/dhcp/agent.py
  - https://opendev.org/openstack/neutron/src/branch/stable/2026.1/neutron/agent/metadata/proxy_base.py
  - https://opendev.org/openstack/neutron/src/branch/stable/2026.1/neutron/notifiers/nova.py
  - https://opendev.org/openstack/kolla-ansible/src/branch/stable/2026.1/ansible/roles/neutron/defaults/main.yml
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Reading the Neutron logs with ML2/OVS. Server, RPC server, OVS, L3, DHCP and metadata agents

Neutron's control plane and its agents log in different places, on different hosts, and an instance's network
problem usually shows in two of them at once: the server that decided, and the agent that should have wired it.
This page says which log holds what, what each says when things are fine, and what the common lines mean. Every
message is the message text from Neutron's 2026.1 source (file named in each table); values in angle brackets are
placeholders.

OVN deployments have no OVS, L3 or DHCP agent; they are covered on their own page (to come). File locations and the
line format: [where the logs are](where-openstack-logs-are-and-how-to-read-a-line.md).

## Which process logs what

Kolla-Ansible 2026.1 runs the Neutron control plane as **three containers**, each with its own log, all enabled by
default (`neutron_services` in the `neutron` role defaults):

| Container | Program | Log (Kolla-Ansible) | What it does |
|---|---|---|---|
| `neutron_server` | the API under uWSGI | `/var/log/kolla/neutron/neutron-server.log`, requests in `neutron-server-uwsgi.log` | REST API: Nova's port create and update (which triggers the port binding), the `network-vif-plugged` notification to Nova |
| `neutron_rpc_server` | `neutron-rpc-server` | `/var/log/kolla/neutron/neutron-rpc-server.log` | RPC with the agents over RabbitMQ: agent heartbeats (`report_state`), device details, port status up/down |
| `neutron_periodic_worker` | `neutron-periodic-workers` | `/var/log/kolla/neutron/neutron-periodic-workers.log` | Periodic jobs, including the agent health check |

On a deployment that runs a single `neutron-server` process (API, RPC and periodic workers together), all three
logs are one file. When a line is not where you expect it, look in the other two.

The agents log on the node where they run (network nodes, computes):
`neutron-openvswitch-agent.log`, `neutron-l3-agent.log`, `neutron-dhcp-agent.log`, `neutron-metadata-agent.log`
under `/var/log/kolla/neutron/` on Kolla-Ansible, or `/var/log/neutron/` (or the journal) on a package install.

Before the logs, the agents' own status is the fastest first read:

```bash
openstack network agent list
openstack port show <port-id> -c status -c binding_host_id -c binding_vif_type
```

`binding_vif_type` `binding_failed` means the server could not bind the port; `ovs` with status `DOWN` means bound
but not (yet) wired by the agent.

## Agent liveness: heartbeats

Each agent reports its state every `report_interval` seconds (default 30, `neutron/conf/agent/common.py`). The
server considers an agent dead when no report arrived for `agent_down_time` seconds (default 75,
`neutron/conf/agent/database/agents_db.py`). A periodic job checks every `agent_down_time // 2` seconds
(`agentschedulers_db.py`).

| Line (message text) | Level | Log | What it means | Next |
|---|---|---|---|---|
| `Agent healthcheck: found <n> dead agents out of <total>:` followed by a table of type, last heartbeat, host | WARNING | periodic workers (or the single server log) | Those agents' heartbeats stopped. The agent process may be down, or alive but unable to reach RabbitMQ, or the RPC server may not be consuming. | the agent's own log on that host; RabbitMQ |
| `<agent type> agent <agent-id> is not active` | WARNING | server | Logged by the L3 scheduler (`get_enabled_agent_on_host`, called from `L3Scheduler.auto_schedule_routers`) when the L3 agent of a host asks for routers but is considered dead. Port binding does not log it. The DHCP scheduler logs its own form, `DHCP agent <agent-id> is not active`. | the agent's log on that host |
| `OVS is down, not reporting state to server` | ERROR | OVS agent | The agent sees `ovs-vswitchd` as dead and **stops sending heartbeats** on purpose, so the server soon marks the OVS agent dead. A common cause of a dead OVS agent whose process is running. | [ovs-vswitchd down](../networking/ovs-vswitchd-down-on-compute.md) |
| `Heartbeat received from <type> agent on host <host>, uuid <uuid> after <delta>, sent at <timestamp>` | INFO | RPC server | Logged only when the agent sets `log_agent_heartbeats = true`. The `after <delta>` is the gap since the previous heartbeat: a delta near `agent_down_time` shows a heartbeat that barely made it. | turn it on for one agent while investigating flapping |
| `Failed reporting state!` with a traceback | ERROR | the agent | The agent could not send its heartbeat, usually a `MessagingTimeout` or a RabbitMQ connection error in the traceback. | RabbitMQ; the RPC server |
| `Agent has just been revived. Doing a full sync.` | INFO | OVS and L3 agents (the DHCP agent logs `Agent has just been revived. Scheduling full sync`) | The server told the agent it was considered dead and is now alive again. The agent resynchronises all its state. Repeating every few minutes: the agent flaps between dead and alive. | heartbeat gaps; RabbitMQ; CPU or memory pressure on the host |

Sources: `neutron/db/agents_db.py`, `neutron/db/agentschedulers_db.py`, `ovs_neutron_agent.py`,
`neutron/agent/l3/agent.py`, `neutron/agent/dhcp/agent.py`.

## neutron-server: port binding

When Nova creates or updates a port with a host (`binding:host_id`), ML2 tries each mechanism driver to bind it on
that host's segment.

| Line (message text) | Level | What it means | Next |
|---|---|---|---|
| `Failed to bind port <port-id> on host <host> for vnic_type <vnic> using segments <segments>` | ERROR | No mechanism driver could bind the port: no live agent of the right type on that host, or the agent's bridge mappings do not cover the network's physical network, or the segment type is not supported there. The port gets `binding_vif_type` `binding_failed`, and Nova fails the build with `Binding failed for port <port-id>, please check neutron logs for more information.` | `openstack network agent list --host <host>`; the agent's `bridge_mappings` against the network's `provider:physical_network` |
| `Failed to bind port <port-id> on host <host> allocated on resource providers: <providers>, because no mechanism driver reports being responsible` | ERROR | Same, for a port with a resource request (QoS minimum bandwidth): the allocation in Placement points to a provider no driver claims. | the agent's resource provider configuration |

Source: `neutron/plugins/ml2/managers.py`; Nova's message: `PortBindingFailed` in `nova/exception.py`.

## neutron-server: telling Nova the port is up

When the agent reports the port up, the server sends `network-vif-plugged` to Nova. The lines, from
`neutron/notifiers/nova.py`:

| Line (message text) | Level | What it means |
|---|---|---|
| `Sending events: [...]` | DEBUG | The batch of events sent to Nova's external events API. |
| `Nova event matching ['<nova-request-id>'] response: {...}` | INFO | Nova accepted the event (code 200). The bracketed value is the list of **Nova's request ids** for the external events call: grep the bare `req-…` id in `nova-api.log` to find the `Creating event network-vif-plugged:<port-id> …` line. |
| `Nova event matching ['<nova-request-id>']: {...} returned with failed status` | WARNING | Nova answered with a non-200 code for this event (for example 404 when the instance is gone). |
| `Failed to notify nova on events: [...]` with a traceback | ERROR | Nova could not be reached: the Keystone credentials of the `[nova]` section, or the Nova API down. |
| `Error response returned from nova: <response>` | ERROR | Nova answered with an error for the batch. |
| `Nova returned NotFound for event: [...]` | DEBUG | The instance no longer exists on Nova's side. |

If Nova waited for `network-vif-plugged` and gave up, check here first whether the event was sent at all: the
[vif_plugged runbook](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md) walks through it.

## OVS agent

### What normal looks like

The OVS agent runs a loop every `polling_interval` seconds (default 2) and logs, at `INFO`, one line per iteration:

```text
Agent rpc_loop - iteration:<n> completed. Processed ports statistics: {...}. Elapsed:<seconds>
```

The `Elapsed` value is the useful part: an iteration that takes much longer than the polling interval means the
agent is slow to wire new ports (many ports, a slow OVSDB, a slow RPC server).

For each port it wires: `Port <port-id> updated. Details: {...}`, then
`Configuration for devices up [<port-id>, ...] and devices down [...] completed.` (both `INFO`).

### Lines that signal a problem

| Line (message text) | Level | What it means | Next |
|---|---|---|---|
| `OVS is dead. OVSNeutronAgent will keep running and checking OVS status periodically.` | WARNING | The agent cannot reach `ovs-vswitchd` / OVSDB. No port can be wired on this host; existing flows may be gone if `ovs-vswitchd` itself is down. | [ovs-vswitchd down](../networking/ovs-vswitchd-down-on-compute.md) |
| `OVS is restarted. OVSNeutronAgent will reset bridges and recover ports.` | WARNING | OVS came back after a restart (the agent's canary flow was missing); the agent rebuilds its flows. A short connectivity gap for this host's instances is expected around this line. | `ovs-vswitchd.log` at the same time |
| `Agent out of sync with plugin!` | INFO | The previous iteration failed or the server asked for a resync; the agent re-reads its ports from the server. Occasional: normal. Every iteration: something fails each time. | the `ERROR` just before it |
| `rpc_loop doing a full sync.` | INFO | A full resynchronisation (after start, revival or error). | — |
| `Port <port> was not found on the integration bridge and will therefore not be processed` | INFO | The agent had seen the port on `br-int`, but it **disappeared** before the agent processed it (deleted, migrated, unplugged): "The port disappeared and cannot be processed" in the source. It does not mean that a new port was never plugged. | usually nothing; the port's later events |

Sources: `ovs_neutron_agent.py`; `polling_interval` default in the OVS agent options.

A tap that was **never plugged** on `br-int` produces no agent line at all: the sign is the *absence* of
`Port <port-id> updated` in the agent log of the port's `binding_host_id`. Check `ovs-vsctl list-ports br-int` and
libvirt on that compute.

Restarting the agent is covered by [restarting the OVS agent does not cut instances](../networking/restarting-the-ovs-agent-does-not-cut-instances.md).

## L3 agent

| Line (message text) | Level | What it means | Next |
|---|---|---|---|
| `Starting router update for <router-id>, action <action>, priority <n>, update_id <id>. Wait time elapsed: <seconds>` | INFO | The agent picked up a router change. `Wait time elapsed` is how long it sat in the agent's queue: large values mean the agent is behind. | — |
| `Finished a router update for <router-id>, update_id <id>. Time elapsed: <seconds>` | INFO | Done. The pair gives the processing time per router. | — |
| `Failed to process compatible router: <router-id>` with a traceback | ERROR (with the router payload at DEBUG) | Configuring the router failed (namespace, iptables, keepalived for HA routers). The agent retries. | the traceback |

Source: `neutron/agent/l3/agent.py`. A router left on a dead agent: [router stuck on dead L3 agent](../networking/router-stuck-on-dead-l3-agent.md).

## DHCP agent

| Line (message text) | Level | What it means | Next |
|---|---|---|---|
| `Synchronizing state` … `Synchronizing state complete` | INFO | A full sync of all networks (at start, after revival or after errors). Long gaps between the two on an agent with many networks are normal. | — |
| `DHCP configuration for ports <ports> is completed` | INFO | The agent wrote the dnsmasq configuration for these ports. A new instance's port should appear here before it asks for an address. | — |
| `Unable to <action> dhcp for <network-id>.` with a traceback | ERROR | Enabling, reloading or disabling dnsmasq for this network failed. Instances on it may get no address. | the traceback; [no IP, DHCP agents down](../networking/new-instance-gets-no-ip-dhcp-agents-down.md) |
| `Unable to sync network state.` with a traceback | ERROR | The full sync failed, often an RPC error. | RabbitMQ; the RPC server |

Source: `neutron/agent/dhcp/agent.py`.

## Metadata agent

| Line (message text) | Level | What it means | Next |
|---|---|---|---|
| `The remote metadata server responded with Forbidden. This response usually occurs when shared secrets do not match.` | WARNING | Nova's metadata API refused the proxied request: `metadata_proxy_shared_secret` differs between the Neutron metadata agent and Nova. | both configurations |
| `Remote metadata server experienced an internal server error.` | WARNING | Nova's metadata API answered 500. | `nova-metadata.log` on Kolla-Ansible (`nova-api.log` where the metadata API runs inside it) |
| `Unexpected error.` with a traceback | ERROR | The agent failed to proxy the request (Nova metadata API unreachable, RPC to the server failed). | the traceback; [no metadata](../networking/new-instance-boots-without-metadata.md) |

Source: `neutron/agent/metadata/proxy_base.py`.

## Reading a network failure in order

1. **Port state** (`port show`): `binding_failed` → server log, binding section; `DOWN` and bound → the agent on
   `binding_host_id`.
2. **Agent list**: a dead agent → its own log on its host, then RabbitMQ.
3. **The agent's log for the port id**: the agent works on ports, not on requests, so the port id, not the request
   id, links this hop.
4. **The server's notification to Nova**, if Nova timed out waiting for `network-vif-plugged`.

## Validity and limits

- Every message on this page is quoted from Neutron 2026.1 source. **None was captured on a lab for this page.**
  A lab request asks for real samples.
- The split into `neutron_server`, `neutron_rpc_server` and `neutron_periodic_worker` is Kolla-Ansible 2026.1's
  default; which process logs a given server-side line follows from where the code runs (API request, RPC handler,
  periodic job) and was not observed.
- ML2/OVS only. Linux bridge, SR-IOV and OVN are not covered.
- The `Nova event matching …` lines show the request ids only when the novaclient response carries them
  (`hasattr(response, 'request_ids')` in the source); otherwise they start with `Nova event`.

## Related

- [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md)
- [Waiting for vif_plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md)
- [ovs-vswitchd down on a compute](../networking/ovs-vswitchd-down-on-compute.md)
- [No IP: DHCP agents down](../networking/new-instance-gets-no-ip-dhcp-agents-down.md)
- [No metadata](../networking/new-instance-boots-without-metadata.md)
- [Logs index](README.md)

---

*Provenance: written by agent:ozarland-author, from Neutron and Kolla-Ansible 2026.1 source code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Freading-neutron-logs-ml2-ovs.md&title=Feedback%3A+logs%2Freading-neutron-logs-ml2-ovs.md):
your release, your deployment method, and what happened.
