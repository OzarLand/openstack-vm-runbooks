---
title: Instance has its address and metadata, but is unreachable from outside
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [neutron, nova, ovs, ovn]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any, self-service (tenant) networks behind a router
  network: ml2/ovs or ml2/ovn
  storage: any
  os: any
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/neutron/2026.1/admin/intro-os-networking.html
  - https://docs.openstack.org/neutron/2026.1/admin/deploy-ovs-selfservice.html
  - https://docs.openstack.org/neutron/2026.1/admin/ovn/routing.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/networking/neutron.html
  - https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/security-group-rule.html
last_verified: 2026-10-22
---

# Instance has its address and metadata, but is unreachable from outside

## Symptom

The instance is `ACTIVE`. It got its fixed address and its metadata: the console log shows the SSH key and the
hostname set by cloud-init. From inside the tenant network, another instance reaches it. From outside the cloud,
nothing answers: no ping, no SSH, a connection timeout rather than a refusal.

```text
$ ssh <user>@<floating-ip>
ssh: connect to host <floating-ip> port 22: Connection timed out
```

If the instance never got an address, or got one but no metadata, see
[new instance gets no IP](new-instance-gets-no-ip-dhcp-agents-down.md) or
[new instance boots without metadata](new-instance-boots-without-metadata.md) instead.

## Impact

Usually one instance or one project: its users cannot reach it. When every floating IP of the cloud stops
answering at once, the cause is shared (the router's gateway node, the external network) and the impact is
platform-wide; check [L3 agent dead, router not moved](router-stuck-on-dead-l3-agent.md) first on ML2/OVS.

## Quick checks

With the project's credentials (or admin), from any host with the OpenStack client:

```bash
# 1. The instance's port, its fixed address, its security groups and port security
openstack port list --server <server-id> -c ID -c "Fixed IP Addresses"
openstack port show <port-id> -c fixed_ips -c security_group_ids -c port_security_enabled

# 2. Ingress rules of each security group on the port
openstack security group rule list --ingress <security-group-id>

# 3. A floating IP on this port?
openstack floating ip list --port <port-id>

# 4. The router of the tenant network: its gateway and its interfaces
openstack port list --network <tenant-network-id> --device-owner network:router_interface -c device_id
# HA or distributed routers use other device owners: list the network's ports without the filter if this is empty
openstack router show <router-id> -c external_gateway_info -c interfaces_info -c status

# 5. The external network is really external
openstack network show <external-network-id> -c router:external -c status
```

Each check rules out one cause. Go through them in order: most cases end at check 2 or 3.

## Diagnosis

### Security groups

Neutron's security groups use a default-deny policy and only contain rules that allow traffic. The `default` group
of each project allows all egress and denies ingress, and an instance started without a security group gets
`default` (Neutron's networking introduction for 2026.1). Check 2 shows the group's actual rules: any ingress rule
there that names a remote security group instead of a prefix admits only members of that group, which explains why
neighbours reach the instance and the outside does not.

In the output of check 2, look for an ingress rule that matches the traffic you test: the protocol (`tcp` for SSH,
`icmp` for ping), the port range, and a remote prefix that includes the client's address. A rule with a remote
group instead of a prefix only admits members of that group.

`port_security_enabled: False` disables security groups on the port altogether; it is then not the cause.

### Floating IP and router

- No floating IP on the port (check 3 empty): the instance has only its tenant address, which is not routed from
  outside. Normal behaviour, not a fault.
- A floating IP exists but is associated with another port, or with another fixed address of a port with several
  (`Fixed IP Address` column): traffic goes elsewhere.
- No router interface on the tenant network (check 4 empty), or a router without `external_gateway_info`: nothing
  routes between the external network and the tenant subnet. Neutron normally refuses to associate a floating IP in
  that situation, so a floating IP that is associated while the path is missing points at a change made afterwards
  (an interface or a gateway removed).

### Where the router lives (only when the API records all look right)

**ML2/OVS with the L3 agent:** the router is a `qrouter-<router-id>` namespace on the network node the L3 agent
scheduled it on.

```bash
openstack network agent list --router <router-id> --long      # which agent, alive, HA state
```

On that node:

**Kolla-Ansible:**

```bash
ip netns exec qrouter-<router-id> ip -brief addr                # the qg- interface carries the floating IP
ip netns exec qrouter-<router-id> iptables -t nat -S | grep -F "<floating-ip>"
```

**Any:** the same commands on the network node that runs `neutron-l3-agent`.

The floating IP must appear on the `qg-` interface and in a DNAT rule towards the fixed address. With DVR, floating
IPs live in the `fip-` namespace of the compute instead; DVR is not covered here.

**ML2/OVN:** the router is a logical router `neutron-<router-id>` in the OVN northbound database, and the floating
IP a `dnat_and_snat` NAT entry on it. North-south traffic goes through the router's active gateway chassis (Neutron
OVN routing guide).

**Kolla-Ansible:**

```bash
docker exec ovn_northd ovn-nbctl lr-nat-list neutron-<router-id>   # a dnat_and_snat line for the floating IP
```

**Any:** `ovn-nbctl lr-nat-list neutron-<router-id>` wherever `ovn-nbctl` reaches the northbound database.

### Inside the guest

If everything above is right, the guest itself may drop the traffic: a host firewall (`ufw`, `firewalld`,
`nftables`) or a service not listening. Look at it from the console (`openstack console url show <server-id>`) or
from a neighbour instance in the tenant network: if a neighbour cannot connect either, while the security groups
allow it, the guest is the cause.

### The external network

If several projects lose their floating IPs at once, check the external network itself: the provider VLAN or flat
network on the network nodes, and the upstream route towards the floating IP range. These are outside OpenStack and
outside this page.

## Possible causes

| Cause | How to tell |
|---|---|
| 1. No security group rule admits the traffic | Check 2: no ingress rule for the protocol, port and source |
| 2. No floating IP, or one associated elsewhere | Check 3 empty, or another port or fixed address |
| 3. No router path: no interface on the tenant network, or no gateway | Check 4 |
| 4. The router exists in the API but not on its node (agent down, NAT missing) | Namespace or OVN NAT checks |
| 5. The guest drops the traffic | All checks pass; a neighbour cannot connect either |
| 6. The external network or upstream routing | Every floating IP affected; nothing wrong inside OpenStack |

## Resolution

### For cause 1: add the missing ingress rule

> ⚠️ **State-changing.** Opens the instance (and every other port using this security group) to the traffic the rule
> describes. Use the narrowest source prefix that works, never `0.0.0.0/0` for administration ports unless the
> project accepts it. Undo: `openstack security group rule delete <rule-id>`.

```bash
# SSH from one admin network
openstack security group rule create --ingress --protocol tcp --dst-port 22 --remote-ip <client-prefix> <security-group-id>
# ICMP (ping) from the same network
openstack security group rule create --ingress --protocol icmp --remote-ip <client-prefix> <security-group-id>
```

Rules are stateful: the reply traffic is allowed without an egress rule.

### For cause 2: associate a floating IP with the right port

> ⚠️ **State-changing.** Moves the floating IP to this port: if it was associated elsewhere, that other port loses it
> and anything using it there is cut off. Undo: associate it back with `openstack floating ip set --port`.

```bash
openstack floating ip create <external-network-id>                         # if none is free
openstack floating ip set --port <port-id> <floating-ip-id>
# for a port with several fixed addresses, choose which one:
openstack floating ip set --port <port-id> --fixed-ip-address <fixed-ip> <floating-ip-id>
```

### For cause 3: give the tenant network a path to the external network

> ⚠️ **State-changing.** Setting a gateway gives every instance of the router's networks outbound access (SNAT) to
> the external network, and consumes an address on it. Adding an interface connects the tenant subnet to the router
> and its other subnets. Undo: `openstack router unset --external-gateway <router-id>`,
> `openstack router remove subnet <router-id> <subnet-id>`.

```bash
openstack router set --external-gateway <external-network-id> <router-id>
openstack router add subnet <router-id> <tenant-subnet-id>
```

### For cause 4: the router is missing on its node

On ML2/OVS, follow [L3 agent dead, router not moved](router-stuck-on-dead-l3-agent.md). On ML2/OVN, a missing NAT
entry while the API shows the floating IP associated points at the synchronisation between Neutron and the OVN
northbound database; this page does not cover repairing it.

### For cause 5: the guest

The instance's owner fixes the guest firewall or the service. Nothing to change in OpenStack.

## Verification

```bash
openstack floating ip list --port <port-id>          # one floating IP, on the right fixed address
openstack security group rule list --ingress <security-group-id>
```

From outside the cloud, from an address inside the prefix you allowed:

```bash
ping -c 3 <floating-ip>
ssh <user>@<floating-ip>
```

## Validity and limits

Written from the Neutron 2026.1 documentation (networking introduction for the security group defaults, the
self-service deployment guide, the OVN routing guide), the Kolla-Ansible 2026.1 Neutron reference (`ovn-nbctl` from
the `ovn_northd` container) and the python-openstackclient 2026.1 source for every option used. Not observed on a
platform yet; a lab run or an operator's confirmation is wanted, in particular of the namespace and OVN checks.

Not covered: DVR, BGP-announced floating IPs, IPv6 without NAT, provider networks attached directly to instances
(no router, no floating IP), and repairing the Neutron to OVN synchronisation.

## Related

- [New instance gets no IP address](new-instance-gets-no-ip-dhcp-agents-down.md)
- [New instance boots without metadata](new-instance-boots-without-metadata.md)
- [L3 agent dead, instances still reachable, router not moved](router-stuck-on-dead-l3-agent.md)
- Neutron 2026.1: [networking introduction](https://docs.openstack.org/neutron/2026.1/admin/intro-os-networking.html),
  [OVN routing](https://docs.openstack.org/neutron/2026.1/admin/ovn/routing.html)
