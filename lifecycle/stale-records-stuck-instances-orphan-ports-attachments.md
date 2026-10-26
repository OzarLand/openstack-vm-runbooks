---
title: Stale records. Instances stuck in BUILD, ERROR or deleting, orphan ports and attachments, phantom allocations
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, placement, neutron, cinder, mariadb]
deployment_methods: [any]
environment:
  topology: any
  network: any
  storage: any
  os: any
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-26, note: "full procedure on a disposable lab except the Cinder parts (no Cinder): stuck build, stuck delete, orphan port, phantom and missing allocations cleaned; archive and purge need --all-cells in nova_api" }
upstream_refs:
  - https://docs.openstack.org/nova/latest/cli/nova-manage.html
  - https://docs.openstack.org/cinder/latest/cli/cinder-manage.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Stale records. Instances stuck in BUILD, ERROR or deleting, orphan ports and attachments, phantom allocations

A long-lived platform accumulates records that describe nothing real: every host failure, interrupted operation and
recovery adds some. They hold quota, distort capacity, and make every investigation start with "is this record
true?". This runbook is the periodic clean-up, using project tools first and the database last.

## Symptom

One or more of:

```text
$ openstack server list --all-projects --status BUILD --long     # instances in BUILD for days
$ openstack server list --all-projects --status ERROR --long
$ openstack server list --all-projects --long | grep deleting     # task_state deleting that never completes
$ openstack port list --device-owner compute:nova --long          # ports whose device_id is a deleted instance
$ openstack volume list --all-projects --status in-use            # attached to instances that no longer exist
$ nova-manage placement audit --verbose                           # allocations for consumers that do not exist
```

Also: quota "in use" higher than what the project actually runs; Placement usage
(`openstack resource provider usage show <rp-uuid>`) disagreeing with the instances the host actually runs.

## Impact

No direct service impact, which is why it is neglected. Indirect: scheduling failures from inflated usage, users
blocked by quota, capacity planning on wrong numbers, slower incident response.

## Quick checks

```bash
# Instances in a transitional state for longer than any operation should take
openstack server list --all-projects --long -c ID -c Status -c "Task State" -c Host \
  | awk '$4=="BUILD" || $4=="ERROR" || $6!="None"'
# (there is no "Updated" column in this list; read it per instance with: openstack server show <server-id> -c updated)

# Placement allocations without a matching instance
nova-manage placement audit --verbose              # read-only without --delete; exit code 3 when it finds some

# Ports whose device no longer exists (port list has no device_id column, so read it per port)
for p in $(openstack port list --device-owner compute:nova -f value -c ID); do
  d=$(openstack port show "$p" -f value -c device_id)
  openstack server show "$d" -f value -c id >/dev/null 2>&1 || echo "port $p: device $d not found"
done

# Volumes attached to a missing instance
openstack volume list --all-projects --status in-use -c ID -c "Attached to" -f value
```

**Kolla-Ansible:** `nova-manage` runs in the `nova_api` container (`sudo docker exec nova_api nova-manage ...`),
`cinder-manage` in `cinder_api`. The `nova_api` container's `[database]` is the cell0 database: `db
archive_deleted_rows` and `db purge` need `--all-cells` there, or they only touch cell0. `virsh` runs in the
`nova_libvirt` container. `openstack resource provider ...` needs the `osc-placement` client plugin.

## Diagnosis

For each stale record, establish what is real before touching it:

- **Instance stuck in `BUILD`/`spawning`**: is there a libvirt domain on the host (`virsh list --all` on the host)?
  Did the build time out? Check `nova-compute.log` for the UUID. Usually safe to delete via the API.
- **Instance stuck in `deleting`**: the compute received the delete and never finished (host died, storage hung).
  Check the host and the domain. If the host is gone, a deletion will complete once Nova considers it `down`
  (local delete), or needs `reset-state` first. A local delete removes the record, the ports and the allocation,
  not the domain: if the host is only stopped, the domain keeps running until `nova-compute` returns and processes
  the delete it had queued.
- **Instance in `ERROR`**: read `fault`. Many are simply failed builds (delete them). Some are real instances whose
  last operation failed (resize, migration): the domain may be running and healthy; reset the state instead of
  deleting.
- **Orphan port**: `device_id` points to a deleted server. Confirm the server is really deleted
  (`openstack server show <id>` returns not found; `nova-manage` shows it archived) before deleting the port.
- **Volume attached to a missing instance**: Cinder has an attachment the instance does not. Compare
  `cinder --os-volume-api-version 3.27 attachment-list --all-tenants 1 --volume-id <volume-id>` with
  `openstack server show` on the instance, if it exists.
- **Phantom allocation**: `placement audit` lists allocations whose consumer (instance) no longer exists, or exists
  on a different host than the allocation says (left over from a failed migration or evacuation). It also lists
  the allocation of an instance still in `BUILD` without a host (for example a build queued for a stopped
  `nova-compute`) as "can be deleted": clear stuck builds first, and check each consumer UUID with
  `openstack server show` before `--delete`.

## Possible causes

Interrupted operations (host failure during build, delete, resize or migration), control-plane outages mid-request,
message-bus incidents that lost RPC calls, manual interventions that bypassed the API. There is rarely a single
cause; the clean-up is routine maintenance.

## Resolution

Work with project tools in this order. Take a database backup before anything that touches the database directly.

### Instances: reset state, then delete through the API

> ⚠️ **State-changing.** `reset-state` only changes Nova's record; it does not touch the hypervisor. Setting
> `--state active` on an instance whose domain does not run gives users a false picture; setting `--state error`
> then deleting is the usual path for a truly dead instance.

```bash
openstack server set --state error --auto-approve <server-id>      # make it deletable
openstack server delete <server-id>
# Instance whose domain is actually running fine after a failed operation:
openstack server set --state active --auto-approve <server-id>
# python-openstackclient 10.x asks for confirmation without --auto-approve; without a terminal it does nothing.
# An instance in BUILD can usually be deleted directly, without the reset.
```

If the delete hangs because the host is gone, Nova performs a local delete once the compute service is `down`; if
the service still reports `up` from a host that lost its storage, fix or fence the host first.

### Placement: audit, then heal or delete

> ⚠️ **State-changing.** `heal_allocations` creates or corrects allocations for existing instances; `audit --delete`
> removes allocations whose consumer does not exist. Both change what the scheduler sees. Run `audit` without
> `--delete` first and read the list.

```bash
nova-manage placement audit --verbose
nova-manage placement audit --verbose --delete
nova-manage placement heal_allocations --verbose
```

### Ports and volumes

> ⚠️ **State-changing.** Deleting a port frees its IP; detaching or resetting a volume attachment changes its
> status. Confirm the owning instance is gone first.

```bash
openstack port delete <port-id>

# Volume still "in-use" for an instance that no longer exists
cinder --os-volume-api-version 3.27 attachment-list --all-tenants 1 --volume-id <volume-id>
#    Not "openstack volume attachment list --volume-id": with python-openstackclient 9.0.0 the filter is ignored
#    and every attachment of the cloud is listed. Check the Volume ID column of whatever you list.
openstack --os-volume-api-version 3.27 volume attachment delete <attachment-id>
```

> 🛑 **Destructive.** On Cinder API versions without attachment objects, the fallback is to force the volume's
> state. It only changes Cinder's record: if a hypervisor still has the volume attached, a second attachment can
> write to it concurrently and corrupt the data. Precondition: confirm on the compute host that no domain uses the
> volume (`virsh domblklist <instance-name>` on the host that last had it; for RBD, `rbd status <pool>/volume-<volume-id>`
> shows no watcher). Cannot be undone if two writers have touched the disk.

```bash
openstack volume set --state available --detached <volume-id>
```

### Archive and purge soft-deleted rows

> 🛑 **Destructive.** `archive_deleted_rows` moves soft-deleted rows to the shadow tables; `purge` then deletes them
> for good. What is lost: the history of deleted instances, volumes and their actions, which audits, billing and
> incident reviews may rely on. It only touches rows already marked deleted; live resources are not affected.
> Precondition: a database backup, and your retention requirements checked (keep the rows the business needs, for
> example with `--before <date>`). Cannot be undone without the backup.

```bash
nova-manage db archive_deleted_rows --until-complete --all-cells
nova-manage db purge --all --all-cells --verbose          # or --before <date>
# 2026.1: --all-cells with --verbose archives, then fails printing the summary
# (ValueError: not enough values to unpack); leave out --verbose. Exit code 1 means rows were archived.
cinder-manage db purge <days>
```

### Direct database surgery, last resort

> 🛑 **Destructive.** Direct edits bypass every consistency check the services perform. Use only when no API or
> `*-manage` command covers the case (for example a `block_device_mapping` row for a volume Cinder does not know).
> Precondition: full database backup, the exact row identified by UUID, one record at a time, verification after
> each. Cannot be undone without the backup.

Prefer an upstream bug report and a project tool over a repeatable SQL fix. If you must, write down the query, the
row before and after, and the reason, in the change record.

## Verification

```bash
openstack server list --all-projects --long | awk '$4=="BUILD" || $6!="None"'   # empty or explainable
nova-manage placement audit --verbose                                         # no orphan allocations
openstack quota show --usage <project>                                        # matches reality
openstack resource provider usage show <rp-uuid>                              # matches what the host runs (osc-placement)
```

## Lab observations

Kolla-Ansible 2026.1 all-in-one, no Cinder. `nova-compute` stopped through its systemd unit, then: a delete left an
instance `ACTIVE`/`deleting`; a create stayed `BUILD`/`scheduling` with no host; an orphan port and a phantom
allocation were made by hand, and one running instance's allocation was deleted.

- The quick checks found all of them. `placement audit` also listed the stuck build's allocation as deletable.
- Once the compute service was reported `down` (about 60 s after it stopped), a second delete completed at once
  (`instance's host ... is down, deleting from database`), but the domain kept running for 3 minutes, until
  `nova-compute` started and processed the queued delete.
- `audit --delete`, `heal_allocations`, `port delete`, the reset to `error` then `active` of a running instance, and
  the delete of the stuck build all worked; quota usage and the provider usage then matched the 3 instances left.
- Archive without `--all-cells` in `nova_api` archived only cell0 (1 instance); with `--all-cells` the 6 deleted
  cell1 instances went to the shadow tables. Volume steps not run (no Cinder).

## Validity and limits

Derived from an operator's clean-up of a long-lived platform; release not recorded. `nova-manage placement audit`
exists since Train (20.0); `heal_allocations` since Rocky (18.0). Command flags change between releases: check
`nova-manage --help` on your platform. Not covered: Octavia, Heat and Magnum resources that hold Nova and Neutron
objects, which have their own clean-up logic.

## Related

- [No valid host with free capacity](../scheduling/no-valid-host-with-free-capacity.md): where phantom allocations show up first.
- [Compute host failure: fence before evacuate](../recovery/compute-host-failure-fence-before-evacuate.md): the main producer of stale records.
- Source: *OpenStack, the Day After Tomorrow* (S. Zaouam), chapter 10, "The state that lies".

---

*Provenance: written by agent:ozarland-author, from the book *OpenStack, the Day After Tomorrow*.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*
