[← OzarLand home](https://github.com/OzarLand/ozarland)

# openstack-vm-runbooks

**Community runbooks for diagnosing and resolving virtual machine problems in OpenStack.**

Part of [OzarLand](https://github.com/OzarLand/ozarland), a collaborative knowledge-sharing space for OpenStack
operators. Share experiences, document solutions, learn from each other. No imposed tools, no single way of
operating OpenStack.

## Scope

Everything an operator does when an instance misbehaves, from the first symptom to the post-fix checks.

**In scope:**

- Instances stuck in an unexpected state (`BUILD`, `deleting`, `resizing`, `migrating`, `ERROR`, `SHUTOFF` when
  it should run, task states that never clear).
- Boot and scheduling failures: `No valid host`, Placement allocation problems, image or flavor issues, boot from
  volume failures, console not available.
- Create, delete and migration errors: live and cold migration, resize, evacuation, shelve and unshelve.
- Instance networking: no DHCP lease, no metadata, no floating IP connectivity, security group surprises, MTU
  symptoms as seen from the instance, port binding failures.
- Volumes: attach and detach failures, stale attachments, volumes stuck `attaching`/`detaching`/`in-use`, boot
  volume problems, multi-attach.
- Diagnostics across Nova, Neutron, Glance, Cinder and Placement with the OpenStack CLI, service logs and the
  hypervisor (libvirt, QEMU).
- Recovery after an incident (compute node loss, storage outage, control plane outage) as it affects instances,
  and the verifications to run afterwards.

**Out of scope:** deploying OpenStack (see `kolla-ansible-deployment-patterns`), upgrading it (see
`openstack-upgrade-playbooks`), and problems that do not surface through instances (control-plane-only,
networking-agent-only, storage-backend-only), which are collected in `ozarland` discussions until a dedicated
runbook repository exists.

## How content is organised

```text
lifecycle/        stuck states, create, delete, resize, shelve
scheduling/       No valid host, Placement, flavors, images, boot from volume
migration/        live and cold migration, evacuation, resize across hosts
networking/       instance connectivity, DHCP, metadata, floating IPs, security groups, port binding
volumes/          attach, detach, stale attachments, boot volumes
diagnostics/      how to read Nova, Neutron, Cinder, Glance, Placement, libvirt logs and states
recovery/         after a compute, storage or control plane incident; post-incident checks
templates/        runbook template
```

Each runbook follows the shared [runbook structure](https://github.com/OzarLand/ozarland/blob/main/docs/conventions/runbook-structure.md):
symptom, impact, quick checks, diagnosis, possible causes, resolution, verification, validity and limits.
State-changing commands carry ⚠️, destructive ones carry 🛑 with what can be lost and what to check first.

A runbook's `status` says how much it has been verified: `draft` (one operator), `validated` (confirmed by someone
else), `experimental` (unconfirmed or possibly outdated). A page written for one deployment method is welcome;
variants for other methods are added by whoever has them.

## Index

| Page | Components | Releases | Status |
|---|---|---|---|
| [Compute disk full. An instance that writes is frozen, while Nova still shows it ACTIVE](lifecycle/compute-disk-full-instance-paused.md) | nova, libvirt, qemu | 2026.1 | draft |
| [I need to restart nova_libvirt on a compute. Its instances keep running](lifecycle/restarting-nova-libvirt-does-not-stop-instances.md) | nova, libvirt | 2026.1 | draft |
| [Instance stuck in BUILD, then ERROR after vif_plugging_timeout](lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md) | nova, neutron, ovn, ovs | 2026.1 | draft |
| [Stale records. Instances stuck in BUILD, ERROR or deleting, orphan ports and attachments, phantom allocations](lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md) | nova, placement, neutron, cinder, mariadb | unrecorded | draft |
| [No valid host was found, although capacity exists](scheduling/no-valid-host-with-free-capacity.md) | nova, placement | unrecorded | draft |
| [CPU steal time on instances with dedicated (pinned) CPUs. Live migration moves the symptom, not the cause](migration/cpu-steal-on-pinned-instances.md) | nova, libvirt, qemu | unrecorded | draft |
| [Moving instances off a compute host for planned maintenance (patching, reboot, hardware)](migration/moving-instances-off-a-host-for-maintenance.md) | nova, libvirt, qemu | unrecorded | draft |
| [Every instance on one compute loses the network while Nova shows them ACTIVE (ovs-vswitchd down)](networking/ovs-vswitchd-down-on-compute.md) | neutron, ovs | 2026.1 | draft |
| [I need to restart the Open vSwitch agent on a compute. Its instances keep their network](networking/restarting-the-ovs-agent-does-not-cut-instances.md) | neutron, ovs | 2026.1 | draft |
| [Instances of one compute answer ping, but large transfers stall](networking/tunnel-mtu-mismatch-large-packets-dropped.md) | neutron, ovs | 2026.1 | draft |
| [L3 agent dead, instances still reachable, router not moved](networking/router-stuck-on-dead-l3-agent.md) | neutron | 2026.1 | draft |
| [New instance has an address, but no SSH key, no user data, default hostname](networking/new-instance-boots-without-metadata.md) | neutron, nova | 2026.1 | lab-verified |
| [New instance is ACTIVE but unreachable, it never got an IP address](networking/new-instance-gets-no-ip-dhcp-agents-down.md) | neutron, nova | 2026.1 | lab-verified |
| [Instance has its address and metadata, but is unreachable from outside](networking/instance-unreachable-from-outside.md) | neutron, nova, ovs, ovn | 2026.1 | draft |
| [Control plane down. What still works for running instances, and what does not](diagnostics/control-plane-down-what-still-works-for-instances.md) | nova, neutron, ovn, ovs, libvirt | unrecorded | draft |
| [Every API call fails at once. Expired TLS certificates or inconsistent Keystone Fernet keys](diagnostics/expired-certificates-and-fernet-keys-break-every-api-call.md) | keystone, haproxy, rabbitmq, mariadb, memcached | unrecorded | draft |
| [The logs of one compute do not line up with the controllers. Its clock has drifted](diagnostics/compute-clock-skew.md) | nova, neutron, ntp | 2026.1 | draft |
| [VM creation fails platform-wide while instances keep running. Check the shared dependencies first](diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md) | nova, neutron, rabbitmq, mariadb, memcached | unrecorded | draft |
| [What a VM creation traverses, and how to trace one failed boot by request id](diagnostics/what-a-vm-creation-traverses-and-tracing-by-request-id.md) | nova, placement, neutron, glance, cinder | unrecorded | draft |
| [Compute host unreachable, instances down. Fence before you evacuate](recovery/compute-host-failure-fence-before-evacuate.md) | nova, libvirt, cinder, ceph | 2026.1 | draft |
| [Reopening provisioning after a control-plane incident. Stop the APIs, recover, verify, reopen](recovery/reopening-provisioning-after-control-plane-incident.md) | nova, neutron, rabbitmq, mariadb, haproxy | unrecorded | draft |
| [nova-compute is down and the compute refuses an SSH login, but the host is alive](recovery/nova-compute-down-host-alive-ssh-refused.md) | nova, libvirt, ssh | 2026.1 | draft |
| [nova-compute is down on a compute, but its instances are still running](recovery/nova-compute-down-instances-still-running.md) | nova, libvirt | 2026.1 | draft |
| [Volume stuck in attaching, reserved or detaching. The attach or detach never finishes](volumes/volume-stuck-in-attaching-or-detaching.md) | cinder, nova, libvirt | 2026.1 | draft |
| [Stale volume attachment. The volume is still in-use for an instance that was deleted or evacuated](volumes/stale-volume-attachment-after-instance-deleted-or-evacuated.md) | cinder, nova, libvirt | 2026.1 | draft |

## Pages wanted next

Candidate runbooks that almost every operator has met and that are not written yet. Each needs a real occurrence
behind it; open a *Problem solved* issue if you have one.

1. Live migration fails: pre-check errors, shared storage assumptions, CPU model mismatch.
2. Floating IP does not answer: where to look between the router, the gateway and the security groups.
3. Instance in `ERROR` after a resize: recovering with `reset-state` and what that does not fix.
4. Console not available: VNC/SPICE proxy and hypervisor side.
5. Instance stuck in `deleting` after the compute host was lost (the deletion-specific details beyond the stale-records page).

## Contributing

- **Ask a question:** [open a Q&A discussion](https://github.com/OzarLand/ozarland/discussions/new?category=q-a) in
  `ozarland`, where all OzarLand discussions live.
- **Share a problem you solved:** open an issue with the
  [*Problem solved* form](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=problem-solved.yml), or a
  pull request with a runbook.
- **Confirm a runbook in your environment:** open a
  [*Documentation proposal* issue](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml)
  and choose "I confirmed an existing page in my environment". This is how runbooks become validated.
- **Add or improve a runbook:** copy `templates/runbook.md`, open a pull request.

Rules that are not optional: no secrets, hostnames, addresses or unredacted logs; flag state-changing and
destructive commands; say where your solution was observed. Full guide: [CONTRIBUTING.md](CONTRIBUTING.md).

## License

All content, documentation and code alike, is licensed under the Apache License 2.0 ([LICENSE](LICENSE)). Reuse it
freely with attribution. See [licensing](https://github.com/OzarLand/ozarland/blob/main/docs/conventions/licensing.md).

OpenStack is a project of the OpenInfra Foundation community. OzarLand is an independent community initiative.
