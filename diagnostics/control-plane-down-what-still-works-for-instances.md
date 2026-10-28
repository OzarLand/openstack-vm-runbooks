---
title: Control plane down. What still works for running instances, and what does not
kind: diagnostic
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, neutron, ovn, ovs, libvirt]
deployment_methods: [any]
environment:
  topology: any
  network: ml2/ovs or ovn; behaviour differs
  storage: any
  os: any
risk: read-only
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-26, note: "full procedure on a disposable lab, ML2/OVS only: control plane stopped 8 min; 0 pings lost, lease renewed, metadata lost, rebooted CirrOS without sshd until return; OVN and L3 HA not tested" }
upstream_refs:
  - https://docs.openstack.org/nova/latest/admin/architecture.html
  - https://docs.openstack.org/neutron/latest/admin/config-dhcp-ha.html
  - https://docs.openstack.org/neutron/latest/admin/ovn/
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Control plane down. What still works for running instances, and what does not

The facts that make "stop the APIs, protect the running workloads" a credible option rather than a euphemism for
an outage. Verify them for your own platform before an incident; they depend on the networking backend.

## What keeps working when the APIs, the bus and the databases are down

- **Running instances** keep running, keep their disks, keep exchanging traffic. The data plane (hypervisors,
  disks, network paths) does not depend on the control plane to carry existing state.
- **Existing DHCP leases** keep being served: with ML2/OVS, by the DHCP agent's `dnsmasq` on the network node as
  long as that host runs; with OVN, DHCP is answered locally on each compute by `ovn-controller`.
- **Router failover** with L3 HA (keepalived) or OVN gateway chassis is a data-plane mechanism and does not need
  the API.
- **Security groups, floating IPs, routes already programmed** stay in effect.

## What is lost

- Creating, modifying, migrating or deleting any resource.
- **Propagation of new state** to the data plane: a new port, a changed security-group rule, a newly associated
  floating IP will not take effect until the control plane returns.
- **Metadata**: an instance that boots or reboots during the outage may fail to reach the metadata service
  (cloud-init keys, hostname, user data), because the metadata agent or proxy needs Nova and Neutron APIs.
- **DHCP for new instances or expired leases** with ML2/OVS, if `neutron-dhcp-agent` cannot reach `neutron-server`
  for new port information (existing leases continue).
- Console access (VNC/SPICE proxies need the API).
- Monitoring that reads the APIs goes blind; the instances are fine, the dashboards are not.

## Two cautions

1. **"Stopping the control plane" means the APIs, the bus and the databases, not the agents.** Neutron agents and
   `ovn-controller` run on the data-plane nodes. Restarting some of them, depending on version and configuration,
   can interrupt traffic briefly (OVS agent restarts may reprogram flows). Leave them alone during a controlled
   stop.
2. **The exact behaviour depends on the backend and the deployment.** Verify on a lab or during a recovery
   exercise: stop the APIs and the bus, then check that an instance keeps its connectivity, renews its lease, and
   that a router failover still works. Write the result down for your platform.
   **Kolla-Ansible 2026.1:** each container is wrapped in a systemd unit `kolla-<container>-container.service` with
   `Restart=always`, so a plain `docker stop` is undone after `RestartSec` (11 s with the Kolla defaults, 59 s on
   the lab, which sets `docker_client_timeout: 600`). Stop the unit instead
   (`systemctl stop kolla-<container>-container.service`) and start it again the same way. This stops the control
   plane: do it on a lab or in a planned recovery exercise only, never to diagnose a live incident.

## How to check on a live platform, read-only

```bash
# From a host that can reach the instance network (or from another instance)
ping -c 3 <instance-address>

# Lease state inside the instance (console or existing session)
ip -4 addr     # valid_lft resets to the lease time after a renewal
cat /var/lib/dhcp/dhclient*.leases 2>/dev/null | tail -5     # dhclient; or networkctl / nmcli; CirrOS uses dhcpcd

# ML2/OVS: dnsmasq still serving on the network node
pgrep -af dnsmasq | grep -F "<network-id>"     # the command line is not always "dnsmasq --no-hosts" (Kolla: "dnsmasq -k --no-hosts")
# OVN: controller still up on the compute, ports still bound
ovs-vsctl show | head; ovn-sbctl show 2>/dev/null       # sbctl needs the southbound DB, which may be up independently

# L3 HA: which node is master for the router
ip netns exec qrouter-<router-id> ip addr | grep -F "<router-external-address>"
```

## Lab observations

Kolla-Ansible 2026.1 all-in-one, ML2/OVS, legacy (non-HA) router. The APIs, conductor, scheduler, Keystone,
Glance, Placement, Heat, Horizon, RabbitMQ, MariaDB and memcached were stopped for about 8 minutes; Neutron agents,
`nova-compute`, libvirt and Open vSwitch were left running.

- A running instance answered 1 ping per second on its fixed and floating addresses with no loss for the whole stop.
- A forced lease renewal inside the instance was answered by dnsmasq (`DHCPREQUEST` / `DHCPACK`) during the stop.
- A metadata request from the running instance got no answer (timeout). An instance rebooted during the stop got its
  address by DHCP but failed every metadata request; CirrOS waited roughly 50 s per try, and did not start `sshd` until
  metadata answered, about a minute after the control plane was started again (8 minutes after its boot).
- When RabbitMQ and `neutron-server` came back, the DHCP agent logged `Agent has just been revived. Scheduling full
  sync` and restarted dnsmasq; no lease was lost.

## Validity and limits

General behaviour of Nova and Neutron with ML2/OVS and OVN as documented upstream and observed by an operator in a
real control-plane incident. The lab confirmation (OpenStack 2026.1, Kolla-Ansible all-in-one) covers ML2/OVS with a
non-HA (legacy) router only: the OVN behaviour, L3 HA and the router or gateway failover check were not tested.
Other backends (Linux Bridge, SR-IOV, vendor SDN) differ.
Octavia load balancers, Designate records and other services with their own data-plane components have their own
behaviour, not covered here.

## Related

- [Reopening provisioning after a control-plane incident](../recovery/reopening-provisioning-after-control-plane-incident.md).
- [VM creation fails platform-wide](vm-creation-fails-platform-wide-check-shared-dependencies.md).
- Source: *OpenStack, the Day After Tomorrow* (S. Zaouam), chapter 1, technical note on control plane and data plane.

---

*Provenance: written by agent:ozarland-author, from the book *OpenStack, the Day After Tomorrow*.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=diagnostics/control-plane-down-what-still-works-for-instances.md&title=Feedback%3A+diagnostics/control-plane-down-what-still-works-for-instances.md):
your release, your deployment method, and what happened.
