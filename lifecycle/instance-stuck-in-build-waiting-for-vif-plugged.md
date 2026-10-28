---
title: Instance stuck in BUILD, then ERROR after vif_plugging_timeout
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, neutron, ovn, ovs]
deployment_methods: [any]
environment:
  topology: any
  network: ovn or ml2/ovs (agent-based)
  storage: any
  os: any
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, all-in-one", date: 2026-10-24, note: "symptom reproduced by pausing the Open vSwitch agent; quick checks, systemctl restart of the paused agent, clean-up and the verification test boot run; OVN and the other causes not run" }
upstream_refs:
  - https://docs.openstack.org/nova/latest/admin/architecture.html
  - https://docs.openstack.org/nova/latest/configuration/config.html#DEFAULT.vif_plugging_timeout
  - https://docs.openstack.org/nova/latest/configuration/config.html#DEFAULT.vif_plugging_is_fatal
  - https://docs.openstack.org/neutron/latest/admin/ovn/
authored_by: agent:ozarland-author
last_verified: 2026-10-24
---

# Instance stuck in BUILD, then ERROR after vif_plugging_timeout

A networking problem wearing a compute symptom. The instance sits in `BUILD` with task state `spawning` for about
five minutes, then goes to `ERROR`. Nova is waiting for Neutron to tell it the port is wired, and the message never
comes.

## Symptom

```text
$ openstack server show <server-id> -c status -c OS-EXT-STS:task_state -c OS-EXT-STS:vm_state
status      | BUILD
task_state  | spawning
vm_state    | building
```

After `vif_plugging_timeout` seconds (default 300), the instance goes to `ERROR`. With `vif_plugging_is_fatal`
(default true) the fault message is:

```text
Build of instance <server-id> aborted: Failed to allocate the network(s), not rescheduling.
```

Its details, and `nova-compute.log` on the selected host, show the chain behind it:

```text
Timeout waiting for [('network-vif-plugged', '<port-id>')] for instance with vm_state building and task_state spawning
... InstanceEventTimeout
... VirtualInterfaceCreateException: Virtual Interface creation failed
... BuildAbortException ... not rescheduling
```

Scheduling itself succeeded: while the instance is in `BUILD` it has a host (`OS-EXT-SRV-ATTR:host`) and an
allocation in Placement. Once the build is aborted, Nova clears the host field: an `ERROR` instance of this kind shows
no host (observed on a lab). The build is not retried on another host.

## Impact

New instances fail on one host, a set of hosts, or everywhere, depending on the cause. Running instances are not
affected. Each failed build leaves an `ERROR` instance that counts in the project's quota until deleted. Its
Placement allocation is held while it waits, then removed when the build is aborted (observed on a lab).

## Quick checks

Run on a controller with admin credentials, unless stated otherwise.

```bash
# 1. Which host was selected, and the fault
openstack server show <server-id> -c OS-EXT-SRV-ATTR:host -c fault
#    Empty host on an ERROR instance: the build was aborted and the host cleared. Find the host in
#    nova-compute.log on the computes (grep <server-id>) or from the other failed builds.

# 2. Is the port bound, and to what?
openstack port list --server <server-id> -c ID -c Status
openstack port show <port-id> -c binding_vif_type -c binding_host_id -c status
#    port list has no "Binding Host" column and silently drops it: read the host from port show
#    binding_vif_type = binding_failed  -> Neutron could not bind the port on that host
#    status = DOWN with a valid vif_type -> bound, but never came up: agent/controller side

# 3. Is the networking agent or OVN controller alive on that host?
openstack network agent list --host <compute-host>

# 4. Is it one host or all of them?
openstack server list --all-projects --status ERROR --long -c ID -c Host -c Created | head
```

## Diagnosis

### Nova (compute host)

`nova-compute.log` for the instance UUID shows the wait: `Preparing to wait for external event network-vif-plugged`,
then the timeout. If instead you see `network-vif-plugged` received and the boot still fails, this is a different
runbook.

**Kolla-Ansible:** `docker logs nova_compute 2>&1 | grep <server-id>` or `/var/log/kolla/nova/nova-compute.log`.
**Any:** `journalctl -u nova-compute | grep <server-id>` or the configured log file.

### Neutron server

Search `neutron-server.log` for the port UUID. Look for the binding attempt and its result:

- `Failed to bind port <port-id> on host <host>`: no mechanism driver accepted the binding. Usual causes: the agent
  on that host is dead or not reporting, the network's segmentation type (VLAN, VXLAN, Geneve) is not configured on
  that host's bridge mappings, or the physical network name does not match.
- Binding succeeded but no `network-vif-plugged` was sent: the agent or OVN controller never reported the port up,
  or Neutron could not reach the Nova API to deliver the event (check `[nova]` section credentials and endpoint in
  `neutron.conf`, and `neutron-server.log` for `Failed to notify nova on events`).

### Agent or OVN controller (compute host)

**ML2/OVS:** `openstack network agent list --host <host>` must show `Open vSwitch agent` alive. An agent whose
process is frozen stops reporting and shows not alive, while its container still looks up.

When the agent stopped reporting matters. Neutron binds the port when the build starts. If the agent was alive then,
the binding succeeds (`binding_vif_type` `ovs`), the port stays `DOWN`, and the build waits for
`vif_plugging_timeout`: this runbook. If the agent was already reported dead, the binding fails at once:
`Failed to bind port` in `neutron-server.log`, `PortBindingFailed` in `nova-compute.log`. Nova then tries another
host, and when none is left the fault is `Exceeded maximum number of retries. Exhausted all hosts available for
retrying build failures`, within seconds, not after five minutes. On a lab, an agent paused 70 s before the request
gave the second case; one paused right before it gave the first.

```bash
# On the compute (Kolla-Ansible)
sudo docker ps -a --filter name=neutron_openvswitch_agent --format '{{.Names}} {{.Status}}'
#    "Up ... (Paused)" means the container is paused; "Exited" means it stopped
```

On the host,
`ovs-vsctl show` should list the port under `br-int`: as `qvo<port-id-prefix>` with the iptables hybrid firewall
driver (the one in the lab's `openvswitch_agent.ini`; the `tap` device then sits on a `qbr` Linux bridge), as
`tap<port-id-prefix>` with the `openvswitch` firewall driver; `neutron-openvswitch-agent.log` shows the
port being processed. An agent that is alive but has a stale or stuck RPC connection (after a RabbitMQ incident, for
example) processes nothing: look for `AMQP server unreachable` or long gaps in the log.

**OVN** *(not run on the lab)*: on the host, `docker logs ovn_controller` (Kolla-Ansible) or `journalctl -u ovn-controller`. Check that
the chassis is registered and claims the port:

```bash
ovn-sbctl show                              # run where ovn-sbctl can reach the southbound DB
ovn-sbctl --columns=logical_port,chassis find Port_Binding logical_port=<port-id>
ovn-nbctl lsp-get-up <port-id>              # true once the port is up
```

An empty `chassis` means the controller on that host never claimed the port: the host's `ovn-bridge-mappings` or
`ovn-encap-ip` external_ids may be wrong, or `ovn-controller` cannot reach the southbound database (check the
`ovn-remote` setting and TLS).

### Message bus (agent-based backends only)

If every host is affected at once after a RabbitMQ or network event, the agents may be connected to a partitioned
or stale bus. See [VM creation fails platform-wide: check the shared dependencies](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md).

## Possible causes

| Cause | How to tell |
|---|---|
| Agent or OVN controller stopped processing after the port was bound | `network agent list` shows it dead now, but the port is bound (`binding_vif_type` not `binding_failed`) and `DOWN`; or `ovn-sbctl show` has no chassis for the host. An agent already dead when the build started gives `binding_failed` instead (see Diagnosis) |
| Bridge mappings or physical network mismatch on the host | `binding_vif_type = binding_failed`; `neutron-server.log` says no mechanism driver bound the port; only networks of one type fail |
| Agents alive but RPC stale after a bus incident | Agents report alive (heartbeat is on a separate connection) but process nothing; recovers on restart |
| Neutron cannot notify Nova | Port is `ACTIVE`, `neutron-server.log` shows notification errors; every host affected |
| Host simply slow (storage, many ports at once) | Event arrives just after the timeout; raising `vif_plugging_timeout` is a workaround, not a fix |

## Resolution

### Agent or controller down, or RPC stale

> ⚠️ **State-changing.** Restarting `neutron-openvswitch-agent` can briefly disrupt data-plane changes on the host
> and, on some versions and configurations, existing flows; restarting `ovn-controller` is usually transparent
> but re-syncs every port on the host. Do it on one host first.

**Kolla-Ansible:** restart the agent through the systemd unit Kolla-Ansible creates for the container (since 2026.1;
a plain `docker restart` bypasses the unit), on the host:
`sudo systemctl restart kolla-neutron_openvswitch_agent-container.service` or
`sudo systemctl restart kolla-ovn_controller-container.service` *(OVN not run on the lab)*. On a paused agent, the unit restart
worked on a lab: it took about 60 s, because the paused container ignores the stop and is killed (`Exited (137)`),
then the agent was alive again.
**Any:** `systemctl restart neutron-openvswitch-agent` or `systemctl restart ovn-controller`.

If `docker ps` shows the agent container `(Paused)`, resume it instead of restarting it:

> ⚠️ **State-changing.** Resumes the frozen agent process; it then processes the ports that changed meanwhile. Undo:
> `docker pause neutron_openvswitch_agent` (which reproduces the failure). Find out what paused it.

```bash
# On the compute (Kolla-Ansible)
sudo docker unpause neutron_openvswitch_agent
```

Builds that already failed are not retried: clean them up as below.

### Bridge mappings or physical network mismatch

*(Not run on the lab, nor is disabling the compute service below.)*

Fix the host's configuration (`bridge_mappings` in the OVS agent config, or `ovn-bridge-mappings` in
`ovs-vsctl get open . external_ids`) so that it matches the other hosts and the network's `provider:physical_network`,
then restart the agent or controller as above. Until it is fixed, keep the scheduler away from the host:

> ⚠️ **State-changing.** Disabling the compute service stops new instances landing on the host. Running instances
> are not affected.

```bash
openstack compute service set --disable --disable-reason "binding_failed on <network>" <compute-host> nova-compute
```

### Neutron cannot notify Nova

*(Not run on the lab.)*

Fix the `[nova]` section of `neutron.conf` (auth URL, credentials, region, endpoint type) and restart
`neutron-server`. Verify with a test boot.

### Clean up the failed instances

> ⚠️ **State-changing.** Deleting the `ERROR` instances frees quota and Placement allocations. Confirm with the
> owner first if the instance may hold data on an attached volume (the volume itself is not deleted unless it was
> created with `delete_on_termination`).

```bash
openstack server delete <server-id>
```

## Verification

```bash
# A test boot on the affected host reaches ACTIVE within the normal time
openstack server create --flavor <flavor> --image <image> --network <network> \
  --availability-zone nova:<compute-host> vif-test
openstack server show vif-test -c status -c OS-EXT-SRV-ATTR:host
openstack port list --server vif-test -c Status      # ACTIVE
openstack server delete vif-test

# No agent or chassis missing
openstack network agent list
```

## Lab observations

Reproduced on a lab: OpenStack 2026.1, Kolla-Ansible 22.2.0, ML2/OVS with VXLAN, Docker, Ubuntu Noble, 3 controllers
and 2 computes. Kolla-Ansible 22.2.0 sets neither `vif_plugging_timeout` nor `vif_plugging_is_fatal`, so Nova's
defaults (300 s, true) applied. The failure was produced by freezing (pausing) the Open vSwitch agent container of
compute-1, then requesting a test instance on that compute.

| | Before | During (360 s) | After (agent resumed) |
|---|---|---|---|
| Open vSwitch agent compute-1 | alive | **not alive** (container `Up (Paused)`) | alive |
| Test instance | none | **fault after 317 s**: `Failed to allocate the network(s), not rescheduling` (`InstanceEventTimeout` → `VirtualInterfaceCreateException`), no host | deleted |

- The fault arrived about 300 s after the request, as `vif_plugging_timeout` predicts, and the build was not tried on
  the other compute.
- The fault message is the `Build of instance … aborted` one; `Virtual Interface creation failed` appears in its
  details and in `nova-compute.log`, not as the message itself. The earlier version of this page said otherwise.
- The host field of the `ERROR` instance was empty.
- An earlier attempt, on a lab that had run many other failures overnight, gave a fault after 134 s without this
  signature. Its message was not read; it is not counted.

Run again on an all-in-one lab (one VM: controller, network and compute), OpenStack 2026.1, Kolla-Ansible 22.2.0,
ML2/OVS, iptables hybrid firewall:

| Agent paused | Port | Result |
|---|---|---|
| 70 s before the request, agent already `XXX` | `binding_failed`, port removed | `ERROR` after **20 s**: `Exceeded maximum number of retries…`; `Failed to bind port` in `neutron-server.log`, `PortBindingFailed` in `nova-compute.log` |
| Right before the request | Bound (`ovs`, `binding_host_id` set), `DOWN` | `BUILD`/`spawning` with a host and a Placement allocation, then `ERROR` after **324 s**: `Build of instance … aborted: Failed to allocate the network(s), not rescheduling`; `Timeout waiting for ['network-vif-plugged-<port-id>']`, `InstanceEventTimeout`, `VirtualInterfaceCreateException` in `nova-compute.log`; host empty; no allocation left |

- A running instance kept answering ping through both (130 of 130 during the first).
- `sudo systemctl restart kolla-neutron_openvswitch_agent-container.service` on the paused agent: 62 s, the container
  killed (`Exited (137)`) and started again, the agent alive 20 s later.
- `openstack server delete` removed both `ERROR` instances. The verification test boot with
  `--availability-zone nova:<compute-host>` reached `ACTIVE` in 16 s with its port `ACTIVE`.

## Validity and limits

The mechanism (Nova waiting for `network-vif-plugged`, bounded by `vif_plugging_timeout`) is stable across recent
releases. It was first written from an operator's description of the provisioning path, release not recorded.

- Observed on one lab build: OpenStack 2026.1, Kolla-Ansible 22.2.0, ML2/OVS (VXLAN), Docker, Ubuntu Noble,
  3 controllers and 2 computes. Cause observed: the Open vSwitch agent of one compute frozen.
- Also observed on an all-in-one lab (see Lab observations): the same fault chain, the quick checks, the binding
  failure of an agent already dead, `systemctl restart` of the paused agent, and the test boot of Verification.
- Observed steps: the symptom and fault chain, the agent shown not alive, resuming the agent (`docker unpause`), and
  deleting the failed instance.
- Not run on a lab: `docker restart` of the agent, every OVN step, the bridge-mapping, stale-RPC, Nova-notification
  and slow-host causes, disabling the compute service.
- OVN commands assume the Neutron OVN driver; ML2/OVS commands assume the OVS agent. Not tested with Linux Bridge or
  SR-IOV ports, where binding failures have additional causes, nor on other releases, other deployment tools or at
  production scale.

## Related

- [No valid host with free capacity](../scheduling/no-valid-host-with-free-capacity.md): the other common build failure, which happens before a host is selected.
- [What a VM creation traverses, and tracing it by request id](../diagnostics/what-a-vm-creation-traverses-and-tracing-by-request-id.md).
- [Restarting the OVS agent does not cut instances](../networking/restarting-the-ovs-agent-does-not-cut-instances.md):
  what a restart of the agent does to running traffic.
- [ovs-vswitchd down on a compute](../networking/ovs-vswitchd-down-on-compute.md)
- Nova: [`vif_plugging_timeout` and `vif_plugging_is_fatal`](https://docs.openstack.org/nova/latest/configuration/config.html#DEFAULT.vif_plugging_is_fatal)
- Source: *OpenStack, the Day After Tomorrow* (S. Zaouam), chapter 5, technical note on the provisioning path.
- Source: lab reproduction by @soufian-zaouam (OpenStack 2026.1, Kolla-Ansible 22.2.0).

---

*Provenance: written by agent:ozarland-author, from the book *OpenStack, the Day After Tomorrow* and a lab reproduction by @soufian-zaouam on 2026.1.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md&title=Feedback%3A+lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md):
your release, your deployment method, and what happened.
