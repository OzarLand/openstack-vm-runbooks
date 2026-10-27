---
title: What a VM creation traverses, and how to trace one failed boot by request id
kind: diagnostic
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, placement, neutron, glance, cinder, keystone, rabbitmq]
deployment_methods: [any]
environment:
  topology: any
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-26, note: "full procedure on a disposable lab: two failed boots (no valid host; port binding to a dead OVS agent) traced by request id across nova, placement and neutron logs" }
upstream_refs:
  - https://docs.openstack.org/nova/latest/admin/architecture.html
  - https://docs.openstack.org/nova/latest/admin/cells.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# What a VM creation traverses, and how to trace one failed boot by request id

A failed VM creation says what the user experienced, not which service caused it. A scheduling problem looks like
a Nova problem, a network timeout like a Neutron problem, a slow operation like a capacity problem, and the
architecture makes each plausible without making any true. Knowing the path, and following one request id through
the logs, replaces guessing by timestamp.

## The path

```text
 user ──► Keystone (token) ──► nova-api ─────────────────────────────► API database (request recorded)
                                  │ RPC (RabbitMQ)
                                  ▼
                             nova-conductor ──► nova-scheduler ──► Placement (candidates, claim allocation)
                                  │                                   cell mapping: which cell DB holds the instance
                                  │ RPC
                                  ▼
                             nova-compute (selected host)
                                  ├──► Glance: image  (or Cinder: boot volume)
                                  ├──► Neutron: create + bind port  ──► agent / ovn-controller wires it
                                  │        ◄── network-vif-plugged event (Neutron ─► nova-api)
                                  └──► libvirt/QEMU: start the domain
```

Where each step fails, and what it looks like:

| Step | Failure shows as | First place to look |
|---|---|---|
| Keystone / memcached | 401, 503, or everything slow | Keystone logs, memcached, certificate expiry |
| nova-api → conductor (RPC) | Request accepted, instance never leaves `scheduling`; `MessagingTimeout` | RabbitMQ health, `nova-api.log` |
| Scheduler / Placement | `ERROR`, fault `No valid host was found`; no host assigned | `nova-scheduler.log`, Placement candidates ([runbook](../scheduling/no-valid-host-with-free-capacity.md)) |
| Cell mapping | `Host '<host>' is not mapped to any cell` | `nova-conductor.log`, `nova-manage cell_v2 list_hosts` |
| Glance | `BUILD`/`spawning` then image download error | `nova-compute.log`, `glance-api.log`, image store backend |
| Cinder (boot from volume) | `BUILD`/`block_device_mapping` stuck; volume stays `creating` | `cinder-volume.log`, storage backend |
| Neutron port create/bind | `binding_failed`; `Failed to bind port` (the user may only see `Exceeded maximum number of retries`) | `neutron-server.log` |
| vif plug on the host | `BUILD`/`spawning` for `vif_plugging_timeout` then `ERROR` | Agent / `ovn-controller` on the host ([runbook](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md)) |
| libvirt / QEMU | `ERROR` with a libvirt error in `fault` | `nova-compute.log`, `libvirtd` log on the host |

Inside Nova the API, conductor, scheduler and compute talk over RabbitMQ. Calls to Keystone, Placement, Glance,
Neutron and Cinder are REST calls through the service catalogue. With agent-based networking, `neutron-server`
talks to its agents over RabbitMQ as well; OVN uses its own databases instead. Every service reads and writes its
own database. A user-visible failure can therefore originate in any service, in the bus, in the database cluster,
or in the network and storage underneath.

## Tracing one request

Every API request carries a request id (`req-<uuid>`) that the services propagate in their logs. Get it, then
follow it.

```bash
# From the failed instance's actions (the request id of the create action)
openstack server event list <server-id>
openstack server event show <server-id> <request-id>       # per-service events and tracebacks (admin)

# Or from the client that made the call: the create's id is on the 202 response; the other ids are lookups
openstack --debug server create ... 2>&1 | grep -E "^RESP: \[202\]" | grep -ioE "x-openstack-request-id: req-[0-9a-f-]+"
```

`server event show` can be thin: after a build failure that was rescheduled and then gave up, it may list only
`compute__do_build_and_run_instance` with result `Success`, and the instance `fault` says `Exceeded maximum number of
retries`. The cause is then only in the logs.

Then search every service's logs for that id, in path order:

**Kolla-Ansible:** logs are under `/var/log/kolla/<service>/` on each node.
```bash
sudo grep -h "<request-id>" /var/log/kolla/nova/nova-api.log /var/log/kolla/nova/nova-conductor.log \
     /var/log/kolla/nova/nova-scheduler.log /var/log/kolla/placement/placement-api.log \
     /var/log/kolla/neutron/neutron-server.log /var/log/kolla/glance/glance-api.log | sort
# and on the selected compute host
sudo grep "<request-id>" /var/log/kolla/nova/nova-compute.log
```
**Any:** `journalctl -u <service> | grep <request-id>`, or the configured log files. A platform whose logs are
centralised and searchable by request id follows one boot across Nova, Placement, Neutron and Glance in minutes.

Note that Neutron and Placement log Nova's request id (the global request id) next to their own for the calls
Nova makes to them (`[req-<nova-id> req-<own-id> ...]`), so the one grep above finds their lines too. Glance, at
the default log level (observed on Kolla-Ansible with `debug = False`), logged no line for a successful call, so finding nothing there does not mean Glance was not
called. A few unrelated lines can carry the id as well (on a lab, a later `GET /v2.1` version request in
`nova-api.log`); check that each line belongs to the request.

## Reading what you find

- The **last service that logged the request id** is where it stopped; the first error after that point is the
  cause, or the symptom of a dependency (bus, database) failing underneath.
- Separate **facts** (observed in logs), **hypotheses** (explain the facts) and **assumptions** (not yet shown).
  Familiar fixes are attractive; a known fix applied to a different failure mode adds risk.
- **Try to disprove** your hypothesis as well as confirm it: can the behaviour be reproduced on one host, with one
  flavor, on one network? Reducing the surface makes every observation worth more.
- If one request fails and others succeed, compare them: same host? same network type? same image? The difference
  is the lead.

## Lab observations

Kolla-Ansible 2026.1 all-in-one, ML2/OVS. Two failed boots traced with the commands above:

- **No valid host** (a flavor larger than the host): Nova's id found in `nova-api`, `placement-api`
  (`GET /allocation_candidates`), `nova-scheduler` (`Got no allocation candidates from the Placement API`) and
  `nova-conductor` (`NoValidHost`), in about 2 s; `server event show` gave the conductor event with the error.
- **Port binding failed** (Open vSwitch agent stopped until reported dead): the same grep found the
  `neutron-server` lines `Refusing to bind port ... to dead agent` and `Failed to bind port ... on host`, with
  Nova's id; `nova-compute.log` had `PortBindingFailed`. The instance went to `ERROR` 15 s after the request with
  the fault `Exceeded maximum number of retries`, and `server event show` reported the compute event as `Success`.

## Validity and limits

Describes the Nova cells v2 architecture (mandatory since Ocata) with the libvirt driver and the Neutron ML2 or OVN
backends. Log paths are given for Kolla-Ansible; other deployments differ. Request-id pairing in logs depends on
release and on `oslo.log` configuration. The lab confirmation (OpenStack 2026.1, Kolla-Ansible all-in-one, ML2/OVS)
traced two failures, no valid host and a port binding failure; it did not reach the Cinder, Glance download,
libvirt and cell-mapping rows of the table.

## Related

- [VM creation fails platform-wide: check the shared dependencies first](vm-creation-fails-platform-wide-check-shared-dependencies.md).
- Source: *OpenStack, the Day After Tomorrow* (S. Zaouam), chapters 5, 8 and 9.

---

*Provenance: written by agent:ozarland-author, from the book *OpenStack, the Day After Tomorrow*.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*
