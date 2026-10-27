---
title: A security group rule has no effect. Traffic it allows is still blocked, or traffic it should block still passes
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [neutron, ovs, ovn, iptables]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any
  network: ml2/ovs (iptables hybrid firewall, Kolla-Ansible's default) or ml2/ovn
  storage: any
  os: any
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/neutron/2026.1/admin/intro-os-networking.html
  - https://docs.openstack.org/neutron/2026.1/admin/config-ovsfwdriver.html
  - https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/security-group-rule.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# A security group rule has no effect. Traffic it allows is still blocked, or traffic it should block still passes

## Symptom

A rule was added to a security group, and the traffic it allows still does not reach the instance (a port stays
closed, ping fails). Or a rule was removed, or never existed, and the traffic still passes. The instance itself is
up, and other traffic works.

## Impact

The instances whose ports use that security group. Traffic that should pass is cut, or traffic that should be filtered
reaches the instance: a security exposure when filtering is expected.

## Quick checks

Read-only. With credentials for the instance's project (or admin):

```bash
# 1. The instance's ports, their security groups, and whether port security is on
openstack port list --server <server-id> -c ID -c "Fixed IP Addresses"
openstack port show <port-id> -c security_group_ids -c port_security_enabled -c allowed_address_pairs

# 2. The rules of each of those groups
openstack security group rule list <security-group-id> --long

# 3. The agents on the instance's compute (ML2/OVS: the Open vSwitch agent; ML2/OVN: the OVN controller)
openstack network agent list --host <compute>
```

Read each rule against the traffic: direction (`ingress` for traffic **to** the instance), ethertype (`IPv4` or
`IPv6`), protocol and port range, and the remote side (`remote_ip_prefix`, or `remote_group_id`: then only traffic from
ports in that group matches).

## Diagnosis

### How Neutron applies security groups (Neutron 2026.1 networking guide)

- A security group is a set of **allow** rules on a port, default deny: traffic matches if any rule of any of the
  port's groups allows it. Groups add up.
- Rules are **stateful**: allowing ingress TCP 22 also allows the return traffic.
- The `default` group of a project allows all egress, and allows ingress **only from ports in `default` itself**
  (IPv4 and IPv6, every protocol and port): Neutron creates it from its default security group rule templates, two
  of which have `remote_group_id` set to the group itself (`neutron/db/securitygroups_db.py`). So instances that share
  `default` reach each other on every port. An admin can change these templates
  (`openstack default security group rule list`). A port created without a group gets `default`.
- Neutron also adds basic and anti-spoofing rules: a port can only send from its own MAC and IP addresses, or those in
  its `allowed_address_pairs`.
- With `port_security_enabled` set to `False` on a port, none of this applies: no security group, no anti-spoofing,
  **everything passes**.

### Where the rules are enforced

- **ML2/OVS with Kolla-Ansible 2026.1**: `openvswitch_agent.ini` sets
  `firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver`. Each port gets a Linux
  bridge (`qbr…`) between the instance and `br-int`, and iptables chains on the compute, named after the port ID
  (its first 10 characters): `neutron-openvswi-i<id>` for ingress, `…-o<id>` for egress, `…-s<id>` for
  anti-spoofing; unmatched traffic goes to `sg-fallback`, which drops it (`neutron/agent/linux/iptables_firewall.py`).
  The Open vSwitch agent writes them; while it is dead, rule changes are not applied on that compute.
- **ML2/OVS with the native driver** (`firewall_driver = openvswitch`): OpenFlow rules in `br-int`. The networking
  guide lists two differences with iptables: traffic that conntrack marks invalid is blocked, and multicast to
  `224.0.0.X` is always allowed.
- **ML2/OVN**: each security group is an OVN port group named `pg_<security group ID, with _ instead of ->` with ACLs;
  ports with port security are also in `neutron_pg_drop`, which drops everything not allowed
  (`neutron/common/ovn/utils.py`). `ovn-northd` and `ovn-controller` on the compute turn them into flows; if either is
  down, changes are not applied.

To see what is enforced:

**Kolla-Ansible, ML2/OVS:** on the compute, inside the agent's container (a host using nftables may not show what
the container wrote),

```bash
sudo docker exec neutron_openvswitch_agent iptables-save | grep -i <first-10-characters-of-port-id>
sudo docker exec neutron_openvswitch_agent ip6tables-save | grep -i <first-10-characters-of-port-id>
```

**Kolla-Ansible, ML2/OVN:** on a host running the OVN northbound database,

```bash
sudo docker exec ovn_nb_db ovn-nbctl --no-leader-only acl-list pg_<security_group_id_with_underscores>
```

**Any:** the same commands without the container prefix.

## Possible causes

### Cause 1: the rule does not match the traffic, or is on another group

The rule is in a group the port does not use; or its direction, ethertype, protocol, port range or remote side does
not match (a common one: `remote_group_id` set, while the traffic comes from outside that group; or an `IPv4` rule for
`IPv6` traffic).

### Cause 2: port security is off on the port

`port_security_enabled` is `False`: no rule applies, and everything passes. Removing a rule changes nothing.

### Cause 3: the change did not reach the compute

ML2/OVS: the Open vSwitch agent on the compute is dead or lagging (quick check 3), so the iptables chains still hold
the old rules. ML2/OVN: `ovn-northd` or the compute's `ovn-controller` is down, so the ACLs are not turned into flows.

### Cause 4: the traffic is filtered elsewhere

The guest's own firewall, a router or its gateway, a floating IP that is not associated, or a physical firewall: the
security group allows it, something else drops it
([unreachable from outside](instance-unreachable-from-outside.md)).

### Cause 5: an allowed address pair or a second port lets it through

Traffic from an address listed in `allowed_address_pairs`, or arriving on another port of the instance with other
groups, is allowed by design.

### Cause 6: both instances are in the `default` group

Traffic that should be blocked passes between two instances of the project because both ports use `default`, which
allows all ingress from its own members (see *How Neutron applies security groups*). Quick checks 1 and 2 show it: an
`ingress` rule with `default` itself as remote group.

## Resolution

### For cause 1: add the rule the traffic needs, to the group the port uses

> ⚠️ **State-changing.** Changes what reaches every port that uses this group, in every instance of the project that
> has it. Undo: `openstack security group rule delete <rule-id>`.

```bash
openstack security group rule create --ingress --ethertype IPv4 --protocol tcp --dst-port <port> \
  --remote-ip <cidr> <security-group-id>
```

Or add a group to the port (`openstack port set --security-group <group> <port-id>`, same warning).

### For cause 2: turn port security back on, if it should be on

> ⚠️ **State-changing.** Filtering starts on that port: traffic not allowed by its groups, or sent from addresses that
> are not the port's own, is dropped at once. Check the groups first, and the allowed address pairs if the instance
> uses other addresses (a VRRP address, a router guest). Undo:
> `openstack port set --no-security-group --disable-port-security <port-id>`.

```bash
openstack port set --enable-port-security --security-group <group> <port-id>
```

### For cause 3: bring the agent back

> ⚠️ **State-changing.** Restarting an agent touches every port on that host; read the warnings in the page linked
> below first.

Start the Open vSwitch agent or `ovn-controller` on the compute, or `ovn-northd`
([port binding failed](port-binding-failed.md), cause 1, for the restarts and their warnings). Once it is back, it
applies the current rules; check with the commands under *Where the rules are enforced*.

### For causes 4 and 5

Fix the filter that drops the traffic, or the pair or port that lets it through.

### For cause 6: give the instances their own group

> ⚠️ **State-changing.** The instance loses the access `default` gave it from its peers, and any rule of `default` it
> relied on. Check the new group's rules first. Undo: `openstack port set --security-group default <port-id>`.

```bash
openstack security group create <group>
openstack security group rule create --ingress --ethertype IPv4 --protocol tcp --dst-port <port> \
  --remote-ip <cidr> <group>
openstack port set --no-security-group --security-group <group> <port-id>
```

Or, as admin, change the templates for groups created from now on (`openstack default security group rule list`,
`create`, `delete`); existing `default` groups keep their rules.

## Verification

From a host allowed by the rule, the traffic passes; from a host not allowed, it does not (for example `nc -zv
<instance-address> <port>` from both). On the compute, the port's chains (ML2/OVS) or the port group's ACLs (ML2/OVN)
show the rule.

## Validity and limits

- Written from Neutron 2026.1 (networking guide on security groups, Open vSwitch native firewall guide,
  `neutron/agent/linux/iptables_firewall.py`, `neutron/common/ovn/utils.py`), Kolla-Ansible 2026.1
  (`openvswitch_agent.ini.j2`), python-openstackclient 2026.1, and the default security group rule templates
  (`neutron/db/securitygroups_db.py`, migration `c33da356b165`). Not reproduced on a lab.
- Connections already established: when a rule is removed, the iptables driver deletes the matching conntrack
  entries (`delete_conntrack_state_by_rule` in `iptables_firewall.py`); with other drivers, notably OVN's stateful
  ACLs, a flow conntrack already tracks may keep passing. Not verified on a lab.
- Not covered: stateless security groups, security group logging, FWaaS.

## Related

- [Instance has its address and metadata, but is unreachable from outside](instance-unreachable-from-outside.md)
- [New instance boots without metadata](new-instance-boots-without-metadata.md): removing the default egress rules
  blocks the metadata service.
- [Port binding failed](port-binding-failed.md)
- Neutron 2026.1: [security groups](https://docs.openstack.org/neutron/2026.1/admin/intro-os-networking.html),
  [native OVS firewall](https://docs.openstack.org/neutron/2026.1/admin/config-ovsfwdriver.html)

---

*Provenance: written by agent:ozarland-author, from the Neutron, Kolla-Ansible and python-openstackclient 2026.1
documentation and source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=networking/security-group-rules-not-applied.md&title=Feedback%3A+networking/security-group-rules-not-applied.md):
your release, your deployment method, and what happened.
