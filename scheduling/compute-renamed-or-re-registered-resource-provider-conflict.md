---
title: A compute was renamed, reinstalled or re-added. nova-compute refuses to start, or gets no resource provider
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [nova, placement]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any multinode
  network: any
  storage: any
  os: any
risk: destructive
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/admin/compute-node-identification.html
  - https://docs.openstack.org/nova/2026.1/admin/troubleshooting/orphaned-allocations.html
  - https://docs.openstack.org/nova/2026.1/cli/nova-manage.html#placement-audit
  - https://docs.openstack.org/osc-placement/latest/
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# A compute was renamed, reinstalled or re-added. nova-compute refuses to start, or gets no resource provider

Nova ties a compute to three records that must agree: the service (by host name), the compute node (by a UUID that
the compute keeps on disk in a `compute_id` file), and the Placement resource provider (same UUID, named after the
hypervisor host name). When the host name changes, the `compute_id` file is lost, or a compute is deleted and added
back under the same name while its old provider stays in Placement, one of them disagrees. Then `nova-compute`
refuses to start, or starts but never gets a resource provider, so nothing is scheduled on it.

## Symptom

One of these in the `nova-compute` log of the compute (**Kolla-Ansible:** `/var/log/kolla/nova/nova-compute.log`).
The messages are from `nova/compute/manager.py` and `nova/scheduler/client/report.py`.

The service exits at start-up, and the container restarts in a loop:

```text
... nova.exception.InvalidConfiguration: My node <node-uuid> has host '<old-host>' but my host is '<new-host>'; Possible rename detected, refusing to start!
```

```text
... nova.exception.InvalidConfiguration: No local node identity found, but this is not our first startup on this host. Refusing to start after potentially having lost that state!
```

```text
... nova.exception.InvalidConfiguration: My hypervisor has existing instances, but I appear to be a new service in this database. Possible database configuration error, refusing to start!
```

Or the service starts and is `up`, but every resource update fails because Placement refuses the provider:

```text
ERROR nova.scheduler.client.report [...] [req-<placement-request-id>] Failed to create resource provider record in placement API for UUID <new-node-uuid>. Got 409: {"errors": [{"status": 409, "title": "Conflict", "detail": "... Conflicting resource provider name: <hypervisor-hostname> already exists. ", ...}]}.
ERROR nova.compute.manager [...] Error updating resources for node <hypervisor-hostname>.
...
nova.exception.ResourceProviderCreationFailed: Failed to create resource provider <hypervisor-hostname>
```

The provider list then shows the name with the *old* UUID, which no hypervisor has any more:

```text
$ openstack resource provider list --name <hypervisor-hostname> -c uuid -c name
| <old-node-uuid> | <hypervisor-hostname> |
$ openstack hypervisor list --long -c ID -c "Hypervisor Hostname" -c State
| <new-node-uuid> | <hypervisor-hostname> | up |
```

On Kolla-Ansible, a deploy can also stop before that, in the `nova-cell` role:

```text
"Found multiple compute services with same host name, please check your Nova compute agents list and fix it. The query with hypervisor name '<nodename>' returned following: [...]"
```

## Impact

- `nova-compute` refusing to start: no new instance, migration or resize on that compute, and the compute's
  service goes `down` after `service_down_time`. Its running instances keep running.
- No resource provider: the compute is `up` but never in the scheduler's candidates. Builds that need it fail with
  `No valid host`, its capacity is lost; nothing else breaks.
- Running instances are not stopped by any of these. They become a problem only if someone "fixes" the compute by
  deleting records while they still run on it.

## Quick checks

Read-only. With admin credentials, plus a shell on the compute.

```bash
# 1. What Nova knows: the service and the compute node for this host, and their state
openstack compute service list --service nova-compute --host <host> --long
openstack --os-compute-api-version 2.53 hypervisor list --long -c ID -c "Hypervisor Hostname" -c State
#    with compute API 2.53 or later, the hypervisor ID is the compute node UUID, which is the provider UUID

# 2. What Placement knows for the same name
openstack resource provider list --name <hypervisor-hostname>

# 3. What the compute itself says it is
hostname; hostname -f
#    Kolla-Ansible: the identity file inside the container, and the one the deploy wrote for it
sudo docker exec nova_compute cat /var/lib/nova/compute_id
sudo cat /etc/kolla/nova-compute/compute_id
#    Any: /var/lib/nova/compute_id (the [DEFAULT] state_path), or compute_id next to nova.conf

# 4. Does anything run there, and does Placement hold allocations on the old provider?
openstack server list --all-projects --host <host> -c ID -c Status
openstack resource provider show --allocations <old-node-uuid>
```

Expected on a healthy compute: one service, one hypervisor whose ID equals the content of `compute_id`, and one
provider with that same UUID and the hypervisor host name.

## Diagnosis

### How Nova identifies a compute (2023.1 and later)

From the upstream *Compute Node Identification* guide and `nova/virt/node.py`:

- At first start, `nova-compute` generates a UUID and writes it to `compute_id` in `[DEFAULT] state_path`
  (`/var/lib/nova`), or uses a `compute_id` file found next to one of its configuration files. It reads this file at
  every start; this UUID is the compute node's identity, and the Placement provider's UUID.
- The service record is still matched by `[DEFAULT] host`, which defaults to the system host name. Kolla-Ansible
  does not set `host` for libvirt computes, so the system host name is what counts.
- At start-up (`init_host` in `nova/compute/manager.py`), `nova-compute` refuses to start when:
  - the compute node of its `compute_id` belongs to another host name (`Possible rename detected`);
  - its service already existed but no `compute_id` file is found (`No local node identity found`);
  - its service is new but libvirt already runs instances (`My hypervisor has existing instances`).
- The upstream guide says the host name must not change after the compute registered, even with the identity file.

### How Kolla-Ansible writes `compute_id`

The `nova-cell` role (`tasks/config.yml`, *Ensure compute_id exists in nova_compute volume*) lists the hypervisors,
matches the one whose name is the host's `ansible_facts.nodename`, and writes its ID to
`/etc/kolla/nova-compute/compute_id`; the container copies it to `/var/lib/nova/compute_id` at start
(`nova-compute.json.j2`). If no hypervisor matches, it writes a UUID derived from the node name. Two consequences:

- a reinstalled compute that keeps its name gets its old UUID back at the next deploy;
- a renamed compute gets a new UUID, so a new compute node and a new provider, while the old ones stay.

### Where the provider name conflict comes from

Placement requires provider names to be unique. A new compute node UUID with the same hypervisor host name as an
existing provider is refused with `409 Conflicting resource provider name`. The old provider is left behind when:

- the compute service was deleted but its provider was not: `openstack compute service delete` deletes the
  provider and its allocations, but ignores a provider that still has allocations afterwards
  (`ResourceProviderInUse` is swallowed in `delete_resource_provider`), and only logs a Placement error
  (`Failed to delete compute node resource provider for compute node <uuid>` in the `nova-api` log);
- the compute's identity changed (a new `compute_id`) while its old compute node and provider stayed.

## Possible causes

| What you see | Cause |
|---|---|
| `Possible rename detected` | Cause 1: the host name changed |
| `My hypervisor has existing instances, but I appear to be a new service` | Cause 1 too: the new name created a new service, while the instances run under the old one |
| `No local node identity found` | Cause 2: the `compute_id` file was lost (reinstall, volume removed) |
| `Conflicting resource provider name`, one provider with the old UUID and no hypervisor with that ID | Cause 3: a provider left over from a deleted or re-identified compute |
| Kolla-Ansible: `Found multiple compute services with same host name` | Cause 3's Nova side: two compute services or hypervisors with that name; check *Quick checks* 1 |

### Cause 1: the host name changed

`hostname` on the compute differs from the `Host` of its service in `compute service list`. Typical after an OS
reinstall with a new name, a change from a short name to an FQDN or back, or a change in DNS or `/etc/hosts` that
changes what `hostname` returns.

### Cause 2: the identity file was lost

Same host name, the service exists, but `compute_id` is missing in the container (Kolla-Ansible: the `nova_compute`
volume was removed or the compute was reinstalled, and the deploy has not run since).

### Cause 3: an old provider with the same name

`openstack resource provider list --name <hypervisor-hostname>` returns a UUID that `hypervisor list` does not have.

## Resolution

### For cause 1: give the compute its old name back

This is the only change that does not touch Nova's records, and the upstream guide does not support renaming a
registered compute.

> ⚠️ **State-changing.** Changes the system host name of the compute. Anything else on the host that uses the name
> (monitoring, certificates, the deployment inventory) sees the old name again. Precondition: the old name is the
> `Host` of the existing service in `compute service list`. Undo: set the new name again.

```bash
sudo hostnamectl set-hostname <old-host>
```

**Kolla-Ansible:** check that the inventory uses that name too, then redeploy the compute so that `compute_id` and
the configuration are written again: `kolla-ansible reconfigure -i <inventory> -t nova --limit <old-host>`.

**Any:** restart `nova-compute`.

If the compute must really have a new name, treat it as a new compute: move its instances off
([maintenance](../migration/moving-instances-off-a-host-for-maintenance.md)), delete its compute service (see
cause 3 for checking that the provider went with it), rename, deploy it as a new compute, and map it with
`nova-manage cell_v2 discover_hosts`.

### For cause 2: put the old identity back

The UUID is the hypervisor ID of this host in *Quick checks* 1 (compute API 2.53 or later).

**Kolla-Ansible:** a deploy writes it back:

> ⚠️ **State-changing.** Rewrites the compute's configuration and restarts its containers. Running instances keep
> running (see [restarting nova_libvirt](../lifecycle/restarting-nova-libvirt-does-not-stop-instances.md)).
> Precondition: one hypervisor in `hypervisor list` with this host's node name. Undo: none needed.

```bash
kolla-ansible reconfigure -i <inventory> -t nova --limit <host>
```

**Any:** write it by hand, then start `nova-compute`.

> ⚠️ **State-changing.** Gives the compute its identity back. Precondition: the UUID is the one of this host's
> hypervisor, not a new one; a wrong UUID makes `nova-compute` take over another compute's records. Undo: remove the
> file.

```bash
openstack --os-compute-api-version 2.53 hypervisor list -f value -c ID -c "Hypervisor Hostname" | grep -w <hypervisor-hostname>
echo -n <node-uuid> | sudo tee /var/lib/nova/compute_id
sudo chown nova:nova /var/lib/nova/compute_id
```

### For cause 3: remove the old provider

First prove it is the leftover: its UUID is not the ID of any hypervisor (*Quick checks* 1), and no instance in
Nova runs on it.

> 🛑 **Destructive.** Deletes the old provider, and before that its allocations. What can be lost: the allocations
> it holds. For an instance that has moved to another compute, only the part on the old provider goes. For an
> instance that still runs on this compute, its only allocation goes: it has none until it is healed on the new
> provider (last step below), and until then the scheduler overcommits the compute. Precondition: every migration
> in a final state and no task state on the instances of this host, and the old provider's allocations saved.
> Undo: none for the provider; the new compute node creates its own provider by itself.

```bash
# save what it holds
openstack resource provider show --allocations <old-node-uuid> -f json > provider-<old-node-uuid>.json

# remove its allocations. The old compute node no longer exists, so audit prints
# "The compute node for UUID <old-node-uuid> can not be found" and lists every consumer on that provider.
sudo docker exec nova_api nova-manage placement audit --verbose --resource_provider <old-node-uuid>
sudo docker exec nova_api nova-manage placement audit --verbose --delete --resource_provider <old-node-uuid>

# then the provider itself
openstack resource provider delete <old-node-uuid>
```

**Any:** the same `nova-manage placement audit` commands where `nova.conf` has `[api_database]` and `[placement]`.
See [Placement allocations out of sync](placement-allocations-out-of-sync-audit-and-heal.md) for what `audit`
reports and its preconditions.

At its next resource update (every 60 seconds by default), `nova-compute` creates its provider: `Created resource
provider record via placement API for resource provider with UUID <new-node-uuid> and name <hypervisor-hostname>.`
Restarting `nova-compute` does the same at once.

Then give the instances that run on this compute their allocations back, on the new provider:

> ⚠️ **State-changing.** Creates allocations from each instance's flavor on the new provider. Precondition: the new
> provider exists (`resource provider list --name <hypervisor-hostname>` shows `<new-node-uuid>`). Undo:
> `openstack resource provider allocation delete <server-id>`.

```bash
sudo docker exec nova_api nova-manage placement heal_allocations --verbose --dry-run
sudo docker exec nova_api nova-manage placement heal_allocations --verbose
```

If there are two compute services for the same host (the Kolla-Ansible assert), delete the one that is `down` and
has no instance, after checking its ID:

> 🛑 **Destructive.** Deletes the compute service record, its compute node, its host mapping, its aggregate
> memberships and its provider. What can be lost: the record of the compute if the wrong ID is taken. Precondition:
> the service is `down`, and `openstack server list --all-projects --host <host>` is empty for it (Nova refuses
> otherwise: `Unable to delete compute service that is hosting instances`). Undo: none; a compute that starts again
> under that name registers anew.

```bash
openstack compute service delete <service-id>
```

## Verification

```bash
# one service, up and enabled
openstack compute service list --service nova-compute --host <host>

# one hypervisor, whose ID is the compute's compute_id
openstack --os-compute-api-version 2.53 hypervisor list --long -c ID -c "Hypervisor Hostname" -c State
sudo docker exec nova_compute cat /var/lib/nova/compute_id

# one provider with that UUID and the hypervisor host name, with its inventory
openstack resource provider list --name <hypervisor-hostname>
openstack resource provider inventory list <node-uuid>

# the compute is a candidate again
openstack allocation candidate list --resource VCPU=1 --resource MEMORY_MB=512 | grep <node-uuid>
```

No more `Error updating resources for node` or `Failed to create resource provider` in the `nova-compute` log at the
next resource update.

## Validity and limits

- Written from Nova 2026.1 (`nova/compute/manager.py`: `init_host`, `_ensure_existing_node_identity`,
  `_check_for_host_rename`, `_sanity_check_new_host`, `_update_available_resource_for_node`;
  `nova/virt/node.py`; `nova/scheduler/client/report.py`: `_create_resource_provider`, `delete_resource_provider`;
  `nova/api/openstack/compute/services.py`: service delete), Placement 2026.1
  (`placement/handlers/resource_provider.py`, the duplicate name message), the upstream *Compute Node
  Identification* guide, and Kolla-Ansible 2026.1 (`nova-cell` role, `tasks/config.yml` and `nova-compute.json.j2`).
  Not reproduced on a lab.
- The log lines are shown with the logger and message from the source; the exception lines are the last line of the
  traceback that the service prints when it exits. The JSON in the Placement 409 is trimmed.
- Not covered: Ironic (no `compute_id`, nodes move between services by design), computes with more than one node,
  and renaming a compute while its instances stay on it, which upstream does not support.

## Related

- [Placement allocations do not match the instances](placement-allocations-out-of-sync-audit-and-heal.md)
- [No valid host was found, although capacity exists](no-valid-host-with-free-capacity.md): an unmapped host, and
  `discover_hosts`.
- [Moving instances off a compute host for planned maintenance](../migration/moving-instances-off-a-host-for-maintenance.md)
- Nova 2026.1: [Compute Node Identification](https://docs.openstack.org/nova/2026.1/admin/compute-node-identification.html)

---

*Provenance: written by agent:ozarland-author, from the Nova, Placement and Kolla-Ansible 2026.1 documentation and
source. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=scheduling/compute-renamed-or-re-registered-resource-provider-conflict.md&title=Feedback%3A+scheduling/compute-renamed-or-re-registered-resource-provider-conflict.md):
your release, your deployment method, and what happened.
