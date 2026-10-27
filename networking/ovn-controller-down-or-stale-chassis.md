---
title: OVN Controller agent down, or listed twice for one host. ovn-controller stopped, or stale Chassis records
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [neutron, ovn, ovs]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any multinode
  network: ml2/ovn
  storage: any
  os: any
risk: destructive
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/neutron/2026.1/admin/ovn/troubleshooting.html
  - https://docs.openstack.org/neutron/2026.1/admin/ovn/index.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# OVN Controller agent down, or listed twice for one host. ovn-controller stopped, or stale Chassis records

## Symptom

With ML2/OVN, `openstack network agent list` shows one of:

- an `OVN Controller agent` (or `OVN Controller Gateway agent`) for a compute with `Alive` `XXX` and `State` `DOWN`;
- **two** OVN agents for the same host, one alive and one down, sometimes under two host names;
- an agent marked `('Chassis' register deleted)`.

And, on the affected compute: new instances stay in `BUILD` and fail (port binding refused, or `vif_plugged` never
arrives), or a new port on a running instance gets no network.

## Impact

- **`ovn-controller` down on a compute:** nothing new is programmed on that host. New ports there cannot be bound
  (`Refusing to bind port <port-id> to dead agent`), so builds and migrations to it fail. Flows already installed in
  Open vSwitch stay in place while `ovn-controller` is stopped, so traffic of existing instances usually continues, but
  any change (a security group rule, a new port, a moved address) is not applied there.
- **Duplicated or stale agents:** a cosmetic problem at first, but a stale gateway chassis can stay a candidate for
  routers' gateway ports, and a stale record can confuse the operators who read the agent list during an incident.

## Quick checks

Read-only.

```bash
# 1. The OVN agents of the host, alive or not
openstack network agent list --host <host> -c ID -c "Agent Type" -c Host -c Alive -c State
```

On the host:

**Kolla-Ansible:**

```bash
sudo docker ps -a --filter name=ovn_controller --format '{{.Names}} {{.Status}}'
sudo tail -n 50 /var/log/kolla/openvswitch/ovn-controller.log
sudo docker exec openvswitch_vswitchd ovs-vsctl get Open_vSwitch . external_ids    # system-id, hostname, ovn-remote
```

On a host running the OVN Southbound database, the chassis registered for that host name:

```bash
sudo docker exec ovn_sb_db ovn-sbctl --no-leader-only --columns=name,hostname list Chassis
sudo docker exec ovn_sb_db ovn-sbctl --no-leader-only --columns=name,nb_cfg_timestamp list Chassis_Private
```

**Any:** the same commands without the container prefix.

The log lines that tell why `ovn-controller` cannot work are described in
[reading the OVN logs](../logs/reading-ovn-logs.md) (database connections, inactivity probes, port claims).

## Diagnosis

From Neutron's OVN troubleshooting guide (2026.1):

- `ovn-controller` registers its host in the Southbound database by creating a **`Chassis`** and a
  **`Chassis_Private`** record, named after the Open vSwitch `system-id`. Neutron builds its OVN agents from these
  records, and considers an agent alive while its `Chassis_Private.nb_cfg_timestamp` keeps being updated.
- When `ovn-controller` stops **gracefully**, it deletes both records. When it is killed, or its host crashes, they
  stay.
- If the `system-id` changes (for example during an upgrade or a reinstall), `ovn-controller` creates new records under
  the new name; the old ones stay, and Neutron shows two agents for the host: one alive, one down. If the host name
  changed too, the two agents show different host names.
- If a decommissioned host's `Chassis` was deleted but not its `Chassis_Private`, the agent shows
  `('Chassis' register deleted)`.

## Possible causes

### Cause 1: `ovn-controller` is stopped or cannot reach the Southbound database

The container is stopped or restarting, or its log shows `connection dropped` or `no response to inactivity probe`
towards the Southbound database or its relay. One agent for the host, down.

### Cause 2: stale records after a crash

The host or `ovn-controller` crashed; it is back, but under a new `system-id`, or the old records were never removed.
Two agents for the host, one down.

### Cause 3: a `system-id` or host name change

An upgrade, a reinstall or a rename changed the `system-id` or the hostname. Two agents, possibly under two names.

### Cause 4: a decommissioned host left records behind

The host is gone; its agent stays down, or is marked `('Chassis' register deleted)`.

## Resolution

### For cause 1: start `ovn-controller` and restore its connection

> ⚠️ **State-changing.** Starts or restarts `ovn-controller` on that host; it recomputes the host's flows from the
> Southbound database. Undo: none needed.

**Kolla-Ansible:** `sudo docker start ovn_controller` (or `docker restart`), then follow its log until it connects.

**Any:** start the `ovn-controller` service.

If it cannot connect, fix the path to the Southbound database or its relay (the `ovn-remote` in `external_ids`, the
network, the database itself). Ports refused while it was down are not rebound automatically: create the instance
again, or run the migration again ([port binding failed](port-binding-failed.md)).

### For causes 2, 3 and 4: remove the stale records

The upstream procedure deletes the stale records from the Southbound database by hand; Neutron then removes the
matching agents, and the `Gateway_Chassis` entries in the Northbound database. After a crash, do it **before**
restarting `ovn-controller` or its host.

> 🛑 **Destructive.** Deletes Southbound records. Deleting the record of a **live** chassis cuts that host off from
> OVN until `ovn-controller` registers it again, and Neutron drops its agent and gateway entries. Precondition: for
> each record, its `name` is not the current `system-id` of any running host (compare with
> `ovs-vsctl get Open_vSwitch . external_ids:system-id` on the host), and its `Chassis_Private.nb_cfg_timestamp` no
> longer moves. Take a backup of the Southbound database first if your procedure allows it. Cannot be undone; a live
> chassis re-registers only when its `ovn-controller` restarts.

```bash
# on a Southbound database host; <stale-name> from the lists above
sudo docker exec ovn_sb_db ovn-sbctl destroy Chassis <stale-name>
sudo docker exec ovn_sb_db ovn-sbctl destroy Chassis_Private <stale-name>
```

With a clustered Southbound database, run the writes against the leader (without `--no-leader-only`).

## Verification

```bash
openstack network agent list --host <host>       # one OVN Controller agent for the host, alive, UP
```

Then a test instance on that host reaches `ACTIVE` with its address
([port binding failed](port-binding-failed.md), *Verification*).

## Validity and limits

- Written from Neutron 2026.1 (OVN troubleshooting guide: *Duplicated or deleted OVN agents*, *Recovering from an OVN
  Chassis crash*) and Kolla-Ansible 2026.1 (container names, log paths). Not reproduced on a lab.
- The statement that existing flows stay in place while `ovn-controller` is stopped is how Open vSwitch keeps
  installed flows; how long traffic really continues has not been measured on an OzarLand lab.
- Not covered: `ovn-northd` and the Northbound/Southbound database clusters themselves, gateway chassis scheduling,
  and the OVN metadata agent.

## Related

- [Reading the OVN logs](../logs/reading-ovn-logs.md)
- [Port binding failed](port-binding-failed.md): a dead OVN agent refuses bindings on its host.
- [Instance stuck in BUILD, waiting for vif_plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md)
- Neutron 2026.1: [OVN troubleshooting](https://docs.openstack.org/neutron/2026.1/admin/ovn/troubleshooting.html)

---

*Provenance: written by agent:ozarland-author, from the Neutron 2026.1 OVN documentation and Kolla-Ansible 2026.1.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=networking/ovn-controller-down-or-stale-chassis.md&title=Feedback%3A+networking/ovn-controller-down-or-stale-chassis.md):
your release, your deployment method, and what happened.
