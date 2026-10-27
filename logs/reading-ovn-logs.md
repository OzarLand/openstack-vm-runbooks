---
title: Reading the OVN logs with Neutron. ovn-controller, ovn-northd, the OVN databases and the Neutron OVN driver
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [ovn, neutron, ovs]
deployment_methods: [kolla-ansible, any]
environment:
  topology: multinode
  network: ovn
  storage: any
  os: any
risk: read-only
confirmed_by: []
upstream_refs:
  - https://github.com/ovn-org/ovn/tree/v25.09.5/controller
  - https://github.com/ovn-org/ovn/blob/v25.09.5/northd/ovn-northd.c
  - https://github.com/openvswitch/ovs/blob/v3.6.0/lib/reconnect.c
  - https://opendev.org/openstack/neutron/src/branch/stable/2026.1/neutron/plugins/ml2/drivers/ovn/mech_driver/mech_driver.py
  - https://opendev.org/openstack/neutron/src/branch/stable/2026.1/neutron/plugins/ml2/drivers/ovn/mech_driver/ovsdb/ovn_client.py
  - https://opendev.org/openstack/kolla-ansible/src/branch/stable/2026.1/ansible/group_vars/all/ovn.yml
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Reading the OVN logs with Neutron. ovn-controller, ovn-northd, the OVN databases and the Neutron OVN driver

With ML2/OVN there are no OVS, L3 or DHCP agents. Neutron writes the wanted state into the OVN Northbound database,
`ovn-northd` translates it into the Southbound database, and `ovn-controller` on each chassis (compute or gateway
node) programs Open vSwitch from it. A port that never comes up has stopped at one of those steps, and each step
has its own log. Messages are quoted from OVN 25.09.5, the Open vSwitch library (`lib/reconnect.c`, 3.6.0) that OVN
daemons use for their database connections, and Neutron 2026.1.

ML2/OVS deployments: [reading the Neutron logs with ML2/OVS](reading-neutron-logs-ml2-ovs.md).

## Where the logs are

**Kolla-Ansible** writes all OVN logs in `/var/log/kolla/openvswitch/` (fluentd's `input/10-openvswitch.conf.j2`
lists them):

| Daemon | Container | Log | Where |
|---|---|---|---|
| ovn-controller | `ovn_controller` | `ovn-controller.log` | every chassis (computes, network nodes) |
| ovn-northd | `ovn_northd` | `ovn-northd.log` | controllers |
| Northbound DB | `ovn_nb_db` | `ovn-nb-db.log` | controllers |
| Southbound DB | `ovn_sb_db` | `ovn-sb-db.log` | controllers |
| Southbound relays | `ovn_sb_db_relay_<n>` | `ovn-sb-relay-<n>.log` | controllers |

Neutron's side is in `/var/log/kolla/neutron/`: `neutron-server.log`, the OVN maintenance worker
(`neutron_ovn_maintenance_worker`) and, on the chassis, `neutron-ovn-metadata-agent.log`.

Kolla-Ansible 2026.1 enables **Southbound relays** with OVN by default (`enable_ovn_sb_db_relay`), one relay per 50
chassis (`ovn_sb_db_relay_compute_per_relay`), listening on ports `1664<n>`. Southbound clients configured with
`ovn_sb_connection`, ovn-controller among them, connect to a relay, not to the Southbound cluster directly
(`group_vars/all/ovn.yml`). neutron-server and ovn-northd are configured with `ovn_sb_connection_no_relay`
(`ml2_conf.ini.j2`, `ovn-northd.json.j2`): they connect to the Southbound cluster on port 6642, so their connection
lines name a different target.

**Format.** OVN and OVS daemons write `<date>T<time>.<ms>Z|<sequence>|<module>|<LEVEL>|<message>`, in **UTC**
(fluentd's parser for these files). `oslo.log` lines in Neutron's logs are in the host's local time: line them up
with care.

```bash
# Kolla-Ansible, on a compute
tail -50 /var/log/kolla/openvswitch/ovn-controller.log
grep -E '\|(WARN|ERR|EMER)\|' /var/log/kolla/openvswitch/ovn-controller.log | tail -20
```

## Database connections (every OVN daemon)

OVN daemons use Open vSwitch's reconnect logic for each database connection, and log its transitions
(`lib/reconnect.c`). `<target>` is the connection string, for example `tcp:<address>:6642` (Southbound),
`tcp:<address>:1664<n>` (a relay), or `unix:/run/openvswitch/db.sock` (the local OVS database).

| Line (message text) | Level | What it means |
|---|---|---|
| `<target>: connecting...` | INFO | Opening the connection. |
| `<target>: connected` | INFO | Connected. |
| `<target>: connection dropped (<error>)` | WARN | The connection broke: the remote database restarted, the network failed, or a TLS problem. |
| `<target>: waiting <n> seconds before reconnect` | INFO | Backing off. The interval grows up to the configured maximum. |
| `<target>: continuing to reconnect in the background but suppressing further logging` | INFO | The backoff reached its maximum; reconnection attempts continue but are no longer logged. Silence after this line does not mean the connection came back: look for `connected`. |
| `<target>: no response to inactivity probe after <n> seconds, disconnecting` | ERR | The remote side did not answer the keep-alive probe: it is overloaded, stuck, or the network drops packets. A classic sign of an overloaded Southbound database or relay. |

```bash
grep -E "connection dropped|inactivity probe|connected$" /var/log/kolla/openvswitch/ovn-controller.log | tail -20
```

## ovn-controller: port binding on the chassis

When an instance's tap device appears on `br-int` with the Neutron port id in its `external_ids:iface-id`, the local
ovn-controller claims the matching logical port. Neutron names the logical switch port after the **Neutron port id**
(`lport_name=port['id']` in `ovn_client.py`), so these lines can be grepped by port id (`controller/binding.c`):

| Line (message text) | Level | What it means |
|---|---|---|
| `Claiming lport <port-id> for this chassis.` | INFO | This chassis takes the port. |
| `<port-id>: Claiming <mac> <ip> ...` | INFO | The addresses claimed with it. |
| `Claiming lport <port-id> for this additional chassis.` | INFO | During a live migration the destination usually claims the port as an additional chassis first. |
| `Changing chassis for lport <port-id> from <old> to <new>.` | INFO | The port moves to this chassis (live migration, or a port bound on two chassis at once). Repeated back and forth between two chassis: both claim it. |
| `Setting lport <port-id> ovn-installed in OVS` | INFO | The flows for the port are installed locally. |
| `Setting lport <port-id> up in Southbound` | INFO | ovn-controller marks the port up; Neutron will see it. |
| `Releasing lport <port-id> from this chassis (sb_readonly=<n>)` | INFO | The port left this chassis (instance deleted or moved). |
| `Setting lport <port-id> down in Southbound` | INFO | The port is marked down. |

```bash
# On the compute that hosts the instance
grep "<port-id>" /var/log/kolla/openvswitch/ovn-controller.log
```

No `Claiming lport` line for a new instance's port means ovn-controller never saw the interface with that
`iface-id`: check `sudo docker exec openvswitch_vswitchd ovs-vsctl --columns=name,external_ids list Interface` on the compute (Kolla-Ansible), and libvirt.

## ovn-northd

Several `ovn-northd` instances run, one active, the others on standby, by holding a lock in the Southbound database
(`northd/ovn-northd.c`):

| Line (message text) | Level | What it means |
|---|---|---|
| `ovn-northd lock acquired. This ovn-northd instance is now active.` | INFO | This instance now translates NB into SB. |
| `ovn-northd lock lost. This ovn-northd instance is now on standby.` | INFO | Another instance took over. Frequent flips mean the instances lose their SB connection. |
| `OVN SB IDL reconnected, force recompute.` | INFO | After a reconnection, a full recomputation. On a large deployment it takes time and CPU. |

```bash
grep -h "lock acquired\|lock lost" /var/log/kolla/openvswitch/ovn-northd.log | tail
```

Exactly one controller should have logged `lock acquired` last.

## The Neutron OVN driver

| Line (message text) | Level | What it means | Next |
|---|---|---|---|
| `Refusing to bind port <port-id> due to no OVN chassis for host: <host>` | WARNING | Neutron found no ovn-controller registered for the host Nova asked for: its `ovn-controller` is down or registered under another hostname. The port ends `binding_failed`. | `openstack network agent list --host <host>`; ovn-controller on that host |
| `Refusing to bind port <port-id> to dead agent: <agent>` | WARNING | The chassis exists but its agent (ovn-controller) is considered dead. | same |
| `Refusing to bind port <port-id> on host <host> due to the OVN chassis bridge mapping physical networks [...] not supporting physical network: <physnet>` | INFO | A flat or VLAN network whose physical network is not in that chassis's `ovn-bridge-mappings`. | the chassis's bridge mappings |
| `OVN reports status up for port: <port-id>` | INFO | Neutron saw the Southbound port go up (after `Setting lport … up in Southbound`). Neutron then notifies Nova (`network-vif-plugged`). | — |
| `OVN reports status down for port: <port-id>` | INFO | The port went down. | — |
| `Maintenance task: …` (inconsistencies found, fixing resource …) | DEBUG / ERROR on failure | The maintenance worker repairs differences between Neutron's database and OVN's Northbound. `Maintenance task: Failed to fix resource …` with a traceback means a resource stays out of sync. | the traceback |

Sources: `mech_driver.py`, `ovsdb/maintenance.py` (Neutron 2026.1).

The chain for a new port, by port id: `Claiming lport <port-id>` (ovn-controller, compute) →
`Setting lport <port-id> up in Southbound` → `OVN reports status up for port: <port-id>` (Neutron) →
`Creating event network-vif-plugged:<port-id> …` (nova-api). The step after the last line you find is where it
stopped; if Nova timed out, see [waiting for vif_plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md).

## Reading an OVN problem in order

1. **Database connections** of the affected ovn-controller: `connection dropped`, `inactivity probe`.
2. **The port on its chassis**: `Claiming lport` and `up in Southbound` for the port id.
3. **Neutron**: `Refusing to bind port …` (binding) or `OVN reports status up` (the event to Nova).
4. **The control plane**: one active `ovn-northd`; the NB/SB database and relay logs for restarts or elections.

## Validity and limits

- Messages are quoted from OVN 25.09.5, Open vSwitch 3.6.0 `lib/reconnect.c` and Neutron 2026.1. The OVN and OVS
  versions in a Kolla 2026.1 image come from its distribution repositories and may differ slightly. **No line was
  captured on a lab.** A lab request asks for real samples.
- Which Neutron process logs `OVN reports status up` (API workers, the maintenance worker) was not observed.
- The NB and SB database servers' own logs (RAFT elections, compaction) are not detailed here.

## Related

- [Reading the Neutron logs with ML2/OVS](reading-neutron-logs-ml2-ovs.md)
- [Waiting for vif_plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md)
- [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md)
- [Logs index](README.md)

---

*Provenance: written by agent:ozarland-author, from OVN, Open vSwitch, Neutron and Kolla-Ansible source code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Freading-ovn-logs.md&title=Feedback%3A+logs%2Freading-ovn-logs.md):
your release, your deployment method, and what happened.
