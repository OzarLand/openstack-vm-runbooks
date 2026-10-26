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
logs/             where the logs are and how to read them: services, infrastructure, hypervisor (see logs/README.md)
recovery/         after a compute, storage or control plane incident; post-incident checks
templates/        runbook template
```

Each runbook follows the shared [runbook structure](https://github.com/OzarLand/ozarland/blob/main/docs/conventions/runbook-structure.md):
symptom, impact, quick checks, diagnosis, possible causes, resolution, verification, validity and limits.
State-changing commands carry ⚠️, destructive ones carry 🛑 with what can be lost and what to check first.

A runbook's `status` says how much it has been verified (see the
[knowledge lifecycle](https://github.com/OzarLand/ozarland/blob/main/docs/knowledge-lifecycle.md)):

| Status | What it means |
|---|---|
| `draft` | Written and reviewed, not yet run on a lab nor confirmed by another operator |
| `lab-verified` | Its steps were run on a lab, on the release and deployment method stated, and behaved as written |
| `validated` | Confirmed by an operator other than its author, in another environment |
| `experimental` | Plausible but unconfirmed, narrow, or possibly outdated |

A page written for one deployment method is welcome; variants for other methods are added by whoever has them.

## Start from what you see

| You see | Start with |
|---|---|
| Every instance on one compute lost the network, Nova still shows them ACTIVE | [ovs-vswitchd down](networking/ovs-vswitchd-down-on-compute.md), then [tunnel MTU mismatch](networking/tunnel-mtu-mismatch-large-packets-dropped.md) |
| Instances answer ping, but large transfers stall | [Tunnel MTU mismatch](networking/tunnel-mtu-mismatch-large-packets-dropped.md) |
| A snapshot stays in `image_uploading`, its image `queued` or `saving` | [Snapshot never finishes](lifecycle/snapshot-stuck-in-image-uploading.md) |
| Every API call fails at once | [Expired certificates or Fernet keys](diagnostics/expired-certificates-and-fernet-keys-break-every-api-call.md) |
| Every API fails, the logs show database errors (`WSREP has not yet prepared node`) | [Galera lost quorum](recovery/galera-cluster-lost-quorum-database-unavailable.md) |
| VM creation fails everywhere, running instances are fine | [Check the shared dependencies first](diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md) |
| A new instance is ACTIVE but never got an IP address | [DHCP agents down](networking/new-instance-gets-no-ip-dhcp-agents-down.md) |
| A new instance has an address but no SSH key, default hostname | [Metadata agents down](networking/new-instance-boots-without-metadata.md) |
| An instance answers from inside its network, not from outside | [Unreachable from outside](networking/instance-unreachable-from-outside.md) |
| An instance stays in BUILD, then ERROR after about 5 minutes | [Waiting for vif_plugged](lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md) |
| A resize stays in VERIFY_RESIZE or RESIZE, or ends in ERROR | [Resize not finished](lifecycle/resize-stuck-in-verify-resize-or-failed.md) |
| "No valid host was found" while computes have room | [No valid host with free capacity](scheduling/no-valid-host-with-free-capacity.md) |
| An instance stays in `deleting`, and deleting it again does nothing | [Stuck in deleting](lifecycle/instance-stuck-in-deleting.md) |
| A new instance is ERROR with "Exceeded maximum number of retries" | [The real error is on the computes that tried](scheduling/build-fails-exceeded-maximum-number-of-retries.md) |
| Placement usage of a compute does not match its instances, or a resource provider cannot be deleted (HTTP 409) | [Placement allocations out of sync](scheduling/placement-allocations-out-of-sync-audit-and-heal.md) |
| A boot from volume ends in ERROR ("Failure prepping block device"), or boots nothing | [Instance fails to boot from volume](scheduling/instance-fails-to-boot-from-volume.md) |
| After a compute reboot, instances stay SHUTOFF or have no network | [After a compute reboot](recovery/after-a-compute-reboot-instances-shutoff-or-without-network.md) |
| A compute is unreachable and its instances are down | [Fence before you evacuate](recovery/compute-host-failure-fence-before-evacuate.md) |
| `MessagingTimeout` in every service, computes reported down while they run | [RabbitMQ partitioned or blocked](recovery/rabbitmq-partition-or-alarm-rpc-timeouts.md) |

## Index

Source: **book** (*OpenStack, the Day After Tomorrow*, the maintainer's book), **lab run** (observed on a
lab), **documentation** (upstream documentation and source), **production incident**.

### Lifecycle

| Page | Components | Releases | Status | Source |
|---|---|---|---|---|
| [Compute disk full. An instance that writes is frozen, while Nova still shows it ACTIVE](lifecycle/compute-disk-full-instance-paused.md) | nova, libvirt, qemu | 2026.1 | lab-verified | lab run |
| [Instance stuck in BUILD, then ERROR after vif_plugging_timeout](lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md) | nova, neutron, ovn, ovs | 2026.1 | lab-verified | book, lab run |
| [Instance stuck in deleting, or in ERROR after a delete. Deleting it again does nothing](lifecycle/instance-stuck-in-deleting.md) | nova, libvirt, cinder, neutron, placement | 2026.1 | draft | documentation |
| [I need to restart nova_libvirt on a compute. Its instances keep running](lifecycle/restarting-nova-libvirt-does-not-stop-instances.md) | nova, libvirt | 2026.1 | lab-verified | lab run |
| [Resize or cold migration not finished. Instance in VERIFY_RESIZE, stuck in RESIZE, or in ERROR after a failed resize](lifecycle/resize-stuck-in-verify-resize-or-failed.md) | nova, placement, libvirt | 2026.1 | draft | documentation |
| [Stale records. Instances stuck in BUILD, ERROR or deleting, orphan ports and attachments, phantom allocations](lifecycle/stale-records-stuck-instances-orphan-ports-attachments.md) | nova, placement, neutron, cinder, mariadb | not recorded ([help wanted](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=lifecycle%2Fstale-records-stuck-instances-orphan-ports-attachments.md&title=Release+for+lifecycle%2Fstale-records-stuck-instances-orphan-ports-attachments.md)) | draft | book |
| [Instance snapshot never finishes. Task state image_pending_upload or image_uploading, image queued or saving](lifecycle/snapshot-stuck-in-image-uploading.md) | nova, glance, libvirt, ceph | 2026.1 | draft | documentation |

### Scheduling

| Page | Components | Releases | Status | Source |
|---|---|---|---|---|
| [New instance in ERROR with "Exceeded maximum number of retries". The real error is on the computes that tried](scheduling/build-fails-exceeded-maximum-number-of-retries.md) | nova, placement | 2026.1 | draft | documentation |
| [Instance fails to boot from volume. ERROR at build, or ACTIVE with no bootable disk](scheduling/instance-fails-to-boot-from-volume.md) | nova, cinder, glance, libvirt | 2026.1 | draft | documentation |
| [No valid host was found, although capacity exists](scheduling/no-valid-host-with-free-capacity.md) | nova, placement | 2026.1 | lab-verified | book |
| [Placement allocations do not match the instances. Compute looks full or empty, audit and heal](scheduling/placement-allocations-out-of-sync-audit-and-heal.md) | nova, placement | 2026.1 | draft | documentation |

### Migration

| Page | Components | Releases | Status | Source |
|---|---|---|---|---|
| [CPU steal time on instances with dedicated (pinned) CPUs. Live migration moves the symptom, not the cause](migration/cpu-steal-on-pinned-instances.md) | nova, libvirt, qemu | 2026.1 | lab-verified | book, lab run |
| [Live migration runs for a long time, then is aborted. The instance writes memory faster than it can be copied](migration/live-migration-never-completes-or-times-out.md) | nova, libvirt, qemu | 2026.1 | draft | documentation |
| [Moving instances off a compute host for planned maintenance (patching, reboot, hardware)](migration/moving-instances-off-a-host-for-maintenance.md) | nova, libvirt, qemu | 2026.1 | lab-verified | book |

### Networking

| Page | Components | Releases | Status | Source |
|---|---|---|---|---|
| [Instance has its address and metadata, but is unreachable from outside](networking/instance-unreachable-from-outside.md) | neutron, nova, ovs, ovn | 2026.1 | lab-verified | documentation, lab run |
| [New instance has an address, but no SSH key, no user data, default hostname](networking/new-instance-boots-without-metadata.md) | neutron, nova | 2026.1 | lab-verified | lab run |
| [New instance is ACTIVE but unreachable, it never got an IP address](networking/new-instance-gets-no-ip-dhcp-agents-down.md) | neutron, nova | 2026.1 | lab-verified | lab run |
| [Every instance on one compute loses the network while Nova shows them ACTIVE (ovs-vswitchd down)](networking/ovs-vswitchd-down-on-compute.md) | neutron, ovs | 2026.1 | draft | lab run |
| [I need to restart the Open vSwitch agent on a compute. Its instances keep their network](networking/restarting-the-ovs-agent-does-not-cut-instances.md) | neutron, ovs | 2026.1 | lab-verified | lab run |
| [L3 agent dead, instances still reachable, router not moved](networking/router-stuck-on-dead-l3-agent.md) | neutron | 2026.1 | lab-verified | lab run |
| [Instances of one compute answer ping, but large transfers stall](networking/tunnel-mtu-mismatch-large-packets-dropped.md) | neutron, ovs | 2026.1 | lab-verified | lab run |

### Volumes

| Page | Components | Releases | Status | Source |
|---|---|---|---|---|
| [Stale volume attachment. The volume is still in-use for an instance that was deleted or evacuated](volumes/stale-volume-attachment-after-instance-deleted-or-evacuated.md) | cinder, nova, libvirt | 2026.1 | draft | documentation |
| [Volume stuck in attaching, reserved or detaching. The attach or detach never finishes](volumes/volume-stuck-in-attaching-or-detaching.md) | cinder, nova, libvirt | 2026.1 | lab-verified | documentation |
| [Volume stuck in creating, or in error right after create](volumes/volume-stuck-in-creating-or-error-after-create.md) | cinder, glance, rabbitmq | 2026.1 | draft | documentation |

### Diagnostics

| Page | Components | Releases | Status | Source |
|---|---|---|---|---|
| [The logs of one compute do not line up with the controllers. Its clock has drifted](diagnostics/compute-clock-skew.md) | nova, neutron, ntp | 2026.1 | draft | lab run |
| [Control plane down. What still works for running instances, and what does not](diagnostics/control-plane-down-what-still-works-for-instances.md) | nova, neutron, ovn, ovs, libvirt | not recorded ([help wanted](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=diagnostics%2Fcontrol-plane-down-what-still-works-for-instances.md&title=Release+for+diagnostics%2Fcontrol-plane-down-what-still-works-for-instances.md)) | draft | book |
| [Every API call fails at once. Expired TLS certificates or inconsistent Keystone Fernet keys](diagnostics/expired-certificates-and-fernet-keys-break-every-api-call.md) | keystone, haproxy, rabbitmq, mariadb, memcached | not recorded ([help wanted](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=diagnostics%2Fexpired-certificates-and-fernet-keys-break-every-api-call.md&title=Release+for+diagnostics%2Fexpired-certificates-and-fernet-keys-break-every-api-call.md)) | draft | book |
| [VM creation fails platform-wide while instances keep running. Check the shared dependencies first](diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md) | nova, neutron, rabbitmq, mariadb, memcached, haproxy, keystone | not recorded ([help wanted](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=diagnostics%2Fvm-creation-fails-platform-wide-check-shared-dependencies.md&title=Release+for+diagnostics%2Fvm-creation-fails-platform-wide-check-shared-dependencies.md)) | draft | book |
| [What a VM creation traverses, and how to trace one failed boot by request id](diagnostics/what-a-vm-creation-traverses-and-tracing-by-request-id.md) | nova, placement, neutron, glance, cinder, keystone, rabbitmq | not recorded ([help wanted](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=diagnostics%2Fwhat-a-vm-creation-traverses-and-tracing-by-request-id.md&title=Release+for+diagnostics%2Fwhat-a-vm-creation-traverses-and-tracing-by-request-id.md)) | draft | book |

### Logs

| Page | Components | Releases | Status | Source |
|---|---|---|---|---|
| [Where the OpenStack logs are, and how to read one log line](logs/where-openstack-logs-are-and-how-to-read-a-line.md) | nova, neutron, placement, keystone, glance, cinder, rabbitmq, mariadb, haproxy, ovs, ovn, libvirt | 2026.1 | draft | documentation |
| [Following one request, instance, port or volume through the logs of every service and host](logs/tracing-one-request-across-services-and-hosts.md) | nova, neutron, placement, cinder, glance, ovs, libvirt | 2026.1 | draft | documentation |

### Recovery

| Page | Components | Releases | Status | Source |
|---|---|---|---|---|
| [After a compute reboot, instances stay SHUTOFF or come back without network](recovery/after-a-compute-reboot-instances-shutoff-or-without-network.md) | nova, neutron, ovs, libvirt | 2026.1 | lab-verified | documentation |
| [Compute host unreachable, instances down. Fence before you evacuate](recovery/compute-host-failure-fence-before-evacuate.md) | nova, libvirt, cinder, ceph | 2026.1 | lab-verified | book, lab run |
| [Every API fails on database errors. The Galera cluster lost quorum, or stopped on every node](recovery/galera-cluster-lost-quorum-database-unavailable.md) | mariadb, galera, kolla-ansible, nova, neutron, keystone | 2026.1 | draft | documentation |
| [nova-compute is down and the compute refuses an SSH login, but the host is alive](recovery/nova-compute-down-host-alive-ssh-refused.md) | nova, libvirt, ssh | 2026.1 | lab-verified | lab run |
| [nova-compute is down on a compute, but its instances are still running](recovery/nova-compute-down-instances-still-running.md) | nova, libvirt | 2026.1 | lab-verified | lab run |
| [RPC timeouts in every service, computes reported down. RabbitMQ is partitioned, blocked by an alarm, or a node is missing](recovery/rabbitmq-partition-or-alarm-rpc-timeouts.md) | rabbitmq, oslo.messaging, kolla-ansible, nova, neutron, cinder | 2026.1 | draft | documentation |
| [Reopening provisioning after a control-plane incident. Stop the APIs, recover, verify, reopen](recovery/reopening-provisioning-after-control-plane-incident.md) | nova, neutron, rabbitmq, mariadb, haproxy | not recorded ([help wanted](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=recovery%2Freopening-provisioning-after-control-plane-incident.md&title=Release+for+recovery%2Freopening-provisioning-after-control-plane-incident.md)) | draft | book |

## Pages wanted next

Candidate runbooks that almost every operator has met and that are not written yet. Each needs a real occurrence
behind it; open a *Problem solved* issue if you have one.

1. Live migration fails: pre-check errors, shared storage assumptions, CPU model mismatch.
2. Floating IP does not answer: where to look between the router, the gateway and the security groups.
3. Console not available: VNC/SPICE proxy and hypervisor side.
4. Instance stuck in `deleting` after the compute host was lost (the deletion-specific details beyond the stale-records page).

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
