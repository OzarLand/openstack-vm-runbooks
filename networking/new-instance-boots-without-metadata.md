---
title: New instance has an address, but no SSH key, no user data, default hostname
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [neutron, nova]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers, 2 computes
  network: ml2/ovs (VXLAN)
  storage: none (no Cinder on the lab)
  os: ubuntu-24.04 (noble), docker
risk: destructive
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/nova/latest/admin/metadata-service.html
  - https://docs.openstack.org/neutron/latest/configuration/metadata-agent.html
  - https://docs.openstack.org/nova/latest/admin/config-drive.html
last_verified: 2026-10-21
---

# New instance has an address, but no SSH key, no user data, default hostname

The instance asks `169.254.169.254` for its metadata at first boot. The request reaches a haproxy in the network's
DHCP namespace, which forwards it to the Neutron metadata agent, which passes it to Nova's metadata API. With every
metadata agent down, the request goes nowhere. The image waits for a while, then boots without its metadata.

## Symptom

- A new instance is `ACTIVE` and has its address, but SSH with its key fails, its user data did not run, and its
  hostname is a default one.
- Instances created earlier are not affected.
- Nova reports nothing: no fault.
- The console log shows the metadata requests failing, then the boot going on without them. On CirrOS:

```text
$ openstack console log show <server-id> | grep -iE 'metadata|169.254|iid|failed'
...
failed 1/20: up ... request failed
...
failed 20/20: up ... request failed
failed to read iid from metadata
...
```

With cloud-init, the data source is reported as not found (not observed on the lab, which ran CirrOS).

## Impact

Every instance created while the agents are down boots without its instance ID, SSH key and user data. It stays
broken after the outage: its first boot, which applies the key and the user data, is over. Running instances do not
need the metadata service to keep working, but were not measured on the lab. Instances booted with a config drive do
not use the metadata service for this (Kolla-Ansible's default does not force a config drive).

## Quick checks

On a controller, with admin credentials, unless stated otherwise.

```bash
# 1. The instance is ACTIVE, with an address, no fault
openstack server show <server-id> -c status -c addresses -c fault

# 2. Its console: metadata requests failed
openstack console log show <server-id> | grep -iE 'metadata|169.254|iid|datasource'

# 3. The metadata agents: are any alive?
openstack network agent list --agent-type metadata

# 4. On each network node: the agent container
docker ps -a --filter name=neutron_metadata_agent --format '{{.Names}} {{.Status}}'
```

If every metadata agent is not alive, this page applies.

## Diagnosis

### Neutron

**Kolla-Ansible:** on a network node, `systemctl status kolla-neutron_metadata_agent-container.service` and
`tail -n 100 /var/log/kolla/neutron/neutron-metadata-agent.log`. If the agent is alive but its log shows errors when
it calls Nova, look at Nova instead.
**Any:** `journalctl -u neutron-metadata-agent`, or the configured log file.

Kolla-Ansible sets `force_metadata = true` for the DHCP agent, so the haproxy that answers `169.254.169.254` runs in
the `qdhcp-` namespace of the network. With the default `neutron_agents_wrappers`, it runs in a container of its own.

```bash
# On the network node that hosts the network
ip netns list | grep qdhcp-<network-id>
```

### Nova

The metadata agent forwards to Nova's metadata API.

**Kolla-Ansible:** on the controllers, `docker ps -a --filter name=nova_metadata --format '{{.Names}} {{.Status}}'`
and the Nova metadata log under `/var/log/kolla/nova/` on the same hosts.

## Possible causes

### Cause 1: every metadata agent is down

`openstack network agent list --agent-type metadata` shows no agent alive. Every new instance that relies on the
metadata service is affected. This is the case the lab reproduced.

### Cause 2: the agents are alive, but Nova's metadata API does not answer

The agents are alive, their log shows errors when calling Nova, and the `nova_metadata` containers are down or
failing. Not reproduced on the lab.

### Cause 3: no DHCP agent for the network

With `force_metadata`, the haproxy lives in the DHCP namespace. If the network has no live DHCP agent, the instance
usually has no address either: see
[New instance is ACTIVE but never got an IP address](new-instance-gets-no-ip-dhcp-agents-down.md).

## Resolution

### For cause 1: start the metadata agents, then find why they stopped

> ⚠️ **State-changing.** Starts the metadata agent on the network node. New instances get their metadata again;
> instances already booted without it are not repaired. Undo: stop the unit again (this reproduces the outage).

**Kolla-Ansible:** on each network node:

```bash
systemctl start kolla-neutron_metadata_agent-container.service
```

**Any:** `systemctl start neutron-metadata-agent` on each network node.

### For the instances created during the outage

A reboot may let the image try again, depending on the image and on how its first-boot tool records what it already
did. It was not tested on the lab.

> ⚠️ **State-changing.** Reboots the instance. Its owner loses whatever runs in it for the time of the boot; nothing
> is lost on disk.

```bash
openstack server reboot <server-id>
```

If the key and the user data are still missing, rebuild the instance from its image.

> 🛑 **Destructive.** Rebuilding replaces the root disk of the instance with a fresh copy of the image: everything
> written to it since it booted is lost. Attached volumes are kept. Run it only on instances created during the
> outage, after their owner confirmed nothing on the root disk matters, or after a snapshot
> (`openstack server image create <server-id>`). It cannot be undone without that snapshot.

```bash
openstack server rebuild --image <image> <server-id>
```

### For cause 2

Fix the Nova metadata API (its containers, its configuration, the shared secret it shares with the metadata agent),
then check the agent log again.

## Verification

```bash
# On a controller: the agents alive
openstack network agent list --agent-type metadata

# A new test instance reads its metadata
openstack server create --flavor <flavor> --image <image> --network <network> --key-name <key> meta-test
openstack console log show meta-test | grep -iE 'instance-id|metadata|iid'
openstack server delete meta-test
```

The console must show the instance ID read from the metadata, and SSH with the key must work.

## Lab observations

The failure was produced by stopping the metadata agent units on the three network nodes of the lab (the
controllers), then requesting a CirrOS test instance on compute-1.

| | Before | During (240 s) | After |
|---|---|---|---|
| Metadata agents | 3 alive | **3 not alive** | 3 alive |
| Test instance | none | **ACTIVE in 16 s**; console: `failed 20/20 … request failed`, `failed to read iid from metadata` | deleted |

- CirrOS waited about 3 minutes for its metadata, then went on booting without it.
- The measurement of running instances' traffic on this run was not valid (it read a leftover of a previous test on
  another host), so running instances were not measured.

## Validity and limits

- Observed on one lab build with the configuration above: OpenStack 2026.1, Kolla-Ansible 22.2.0, ML2/OVS with VXLAN,
  Docker, Ubuntu Noble, 3 controllers and 2 computes, CirrOS test instance, no config drive.
- Observed: the symptom and signature, and the agents alive again after their units were started.
- Not run: rebooting or rebuilding an instance created during the outage, a new instance reading its metadata after
  the restart (the test instance was deleted), running instances during the outage. Causes 2 and 3 were not
  reproduced.
- Not tested: cloud-init images, config drive, OVN (its metadata agent is different), other releases, other
  deployment tools, production scale.

## Related

- [New instance is ACTIVE but never got an IP address](new-instance-gets-no-ip-dhcp-agents-down.md): the same
  "ACTIVE but broken" pattern, with the DHCP agents.
- [Router stuck on a dead L3 agent](router-stuck-on-dead-l3-agent.md)
- [What a VM creation traverses, and tracing it by request id](../diagnostics/what-a-vm-creation-traverses-and-tracing-by-request-id.md)
- Nova: [Metadata service](https://docs.openstack.org/nova/latest/admin/metadata-service.html),
  [Config drives](https://docs.openstack.org/nova/latest/admin/config-drive.html)
- Neutron: [metadata agent configuration](https://docs.openstack.org/neutron/latest/configuration/metadata-agent.html)
- Source: lab reproduction by the OzarLand maintainer (OpenStack 2026.1, Kolla-Ansible 22.2.0).
