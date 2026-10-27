---
title: nova-conductor or nova-scheduler is down. What still works, what waits, what fails, and where failed builds go (cell0)
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, rabbitmq]
deployment_methods: [kolla-ansible, any]
environment:
  topology: multinode, 3 controllers
  network: any
  storage: any
  os: any
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, 3 controllers + 2 computes, RabbitMQ cluster with quorum queues", date: 2026-10-27, note: "nova_conductor, then nova_scheduler, stopped on all three controllers (systemctl stop); queued build ACTIVE 32 s after the conductor restart, computes up after 11 s; scheduler-down build ERROR in 61 s with fault MessagingTimeout; cell0 cleanup run; a start sent while the conductor was down left task_state powering-on, reset with --auto-approve --state error and a hard reboot; not run: resize, migrations, evacuate, a stop or reboot of a running instance, a local delete, cell0 database unreachable, --state active" }
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, 3 controllers + 2 computes, RabbitMQ cluster with quorum queues", date: 2026-10-27, note: "nova_conductor, then nova_scheduler, stopped on all three controllers (systemctl stop); queued build ACTIVE 32 s after the conductor restart, computes up after 11 s; scheduler-down build ERROR in 61 s with fault MessagingTimeout; cell0 cleanup run; a start sent while the conductor was down left task_state powering-on, reset with --auto-approve --state error and a hard reboot; not run: resize, migrations, evacuate, a stop or reboot of a running instance, a local delete, cell0 database unreachable, --state active" }
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/admin/architecture.html
  - https://docs.openstack.org/nova/2026.1/admin/cells.html
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#DEFAULT.long_rpc_timeout
  - https://docs.openstack.org/api-guide/compute/down_cells.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# nova-conductor or nova-scheduler is down. What still works, what waits, what fails, and where failed builds go (cell0)

`nova-conductor` and `nova-scheduler` run on the controllers and talk to everything else over RabbitMQ. When every
copy of one of them is down, Nova does not fail at once and does not fail everywhere: some requests wait in a queue
and finish later, some wait for a long timeout and then fail, and running instances do not notice. This page says
which is which, what the logs show, and where the instances that failed to schedule end up: in the `cell0`
database.

## Symptom

### nova-conductor down on every controller

- New instances stay in `BUILD` with task state `scheduling`, without error.
- Within a minute, every compute is reported `down` by `openstack compute service list`, while its instances keep
  running. Each `nova-compute` log says once (`nova/servicegroup/drivers/db.py`):

  ```text
  WARNING nova.servicegroup.drivers.db [-] Lost connection to nova-conductor for reporting service status.: oslo_messaging.exceptions.MessagingTimeout: Timed out waiting for a reply to message ID <message-id>
  ```

- A `nova-compute` that starts while the conductors are down repeats (`nova/conductor/api.py`):

  ```text
  WARNING nova.conductor.api [...] Timed out waiting for nova-conductor.  Is it running? Or did this service start before nova-conductor?  Reattempting establishment of nova-conductor connection...: oslo_messaging.exceptions.MessagingTimeout: Timed out waiting for a reply to message ID <message-id>
  ```

  about every 60 s, until `INFO nova.conductor.api [...] nova-conductor connection established successfully`.

- In RabbitMQ, the `conductor` queue grows and has no consumer (on the lab: 3, then 6 messages, 0 consumers).

### nova-scheduler down on every controller

- New instances stay in `BUILD` / `scheduling` for about a minute (`rpc_response_timeout`), then go to `ERROR`.
  Their fault is the exception's class name, not `No valid host` (`exception_to_dict` in `nova/compute/utils.py`
  keeps only the class name of a non-Nova exception):

  ```text
  $ openstack server show <server-id> -c status -c fault
  | fault  | {'code': 500, 'message': 'MessagingTimeout', ...} |
  | status | ERROR |
  ```

- The conductor log has the failure and its traceback (`nova/conductor/manager.py`):

  ```text
  ERROR nova.conductor.manager [...] Failed to schedule instances: oslo_messaging.exceptions.MessagingTimeout: Timed out waiting for a reply to message ID <message-id>
  ERROR nova.conductor.manager   File ".../nova/conductor/manager.py", line 1677, in schedule_and_build_instances
  ...
  ```

- In RabbitMQ, the `scheduler` queue has no consumer. `openstack compute service list` still shows the schedulers `up`
  until their heartbeat is older than `service_down_time`.

## Impact

| | conductor down | scheduler down |
|---|---|---|
| Running instances | unaffected | unaffected |
| `openstack server list`, `show`, API reads | work (nova-api reads the databases itself) | work |
| New instances | wait in `BUILD`, finish when conductor is back | fail after about `rpc_response_timeout` (60 s), `ERROR` in cell0 |
| Resize, cold migration, live migration without a host, evacuate | wait in the queue | wait, then fail |
| Start, stop, reboot of existing instances | accepted by the API and sent to the compute, which needs the conductor for its database updates: they time out (a start: `MessagingTimeout` in `nova-compute.log` about 2 min later on the lab) and can leave a `task_state` set (see *Resolution*, *A task state left set*) | work |
| Compute services | reported `down` after `service_down_time` (60 s) | stay `up` |
| Danger | deleting an instance while its compute is reported `down` is a local delete (see Diagnosis) | none beyond failed requests |

## Quick checks

Read-only. On a controller.

```bash
# 1. Are the containers running?  (Kolla-Ansible; nova_super_conductor exists only with enable_cells)
sudo docker ps -a --filter name=nova_conductor --filter name=nova_scheduler --filter name=nova_super_conductor \
  --format '{{.Names}} {{.Status}}'

# 2. Do their queues have consumers? (<vhost> is the one in transport_url, "/" by default on Kolla-Ansible)
sudo docker exec rabbitmq rabbitmqctl list_queues -p <vhost> name messages consumers | grep -E '^(conductor|scheduler)\s'
#    expected: consumers > 0 on both (\s skips the per-host conductor.<host> and scheduler.<host> queues).
#    0 consumers and messages growing on conductor: no conductor listening.

# 3. Their logs (Kolla-Ansible)
sudo tail -n 50 /var/log/kolla/nova/nova-conductor.log
sudo tail -n 50 /var/log/kolla/nova/nova-scheduler.log

# 4. With admin credentials: services as Nova sees them
openstack compute service list
```

`openstack compute service list` shows `nova-conductor` and `nova-scheduler` rows too, with their own heartbeat:
`down` there is the quickest confirmation, but it says nothing about why.

## Diagnosis

### What the conductor does, so what stops without it

- `nova-api` casts new builds to the conductor (`schedule_and_build_instances`). A cast does not wait: the API returns
  `202` and the request sits in the `conductor` queue until a conductor consumes it.
- `nova-compute` has no database access of its own; its database reads and writes, including the service heartbeat,
  go through the conductor. Without a conductor, computes stop reporting and are seen `down` after
  `[DEFAULT] service_down_time` (60 s), and any compute operation that saves the instance stalls.
- Resize, migrations, evacuate and rebuild are also cast to the conductor (`ComputeTaskAPI`).

When a conductor comes back, it consumes the queued requests, and the computes log `Recovered from being unable to
report status.` (on the lab: computes `up` 11 s and the queued build `ACTIVE` 32 s after the restart).

### What the scheduler does, so what fails without it

The conductor *calls* the scheduler (`select_destinations`) and waits for the answer. `nova/scheduler/rpcapi.py`
prepares this call with `timeout=CONF.long_rpc_timeout` (1800 s) and `call_monitor_timeout=CONF.rpc_response_timeout`
(60 s). With a call monitor, oslo.messaging raises `MessagingTimeout` as soon as no heartbeat has come for
`call_monitor_timeout`, and only a scheduler that picked up the message sends heartbeats. `long_rpc_timeout` bounds a
call that a live scheduler is working on. So with no scheduler consuming, a build fails after about 60 s: the
conductor logs `Failed to schedule instances` and buries the instance in cell0 (`_bury_in_cell0` in
`nova/conductor/manager.py`): it creates the instance record
in the cell0 database with `vm_state` `error` and the exception as fault.

### cell0, where failed builds live

Every instance that failed before it reached a compute (no valid host, scheduler timeout, a quota or image error at
scheduling) lives in the `cell0` database, not in the cell database of the computes. It has no host and nothing to
clean up on any compute. `openstack server list` shows it like any other instance. `nova-manage cell_v2 list_cells
--verbose` shows cell0's database connection.

If the cell0 database itself is unreachable, `nova-api` treats it as a down cell: with the default
`[api] list_records_by_skipping_down_cells = True`, listings skip it (and from compute API 2.69 return partial
records for its instances) instead of failing.

### A local delete while computes are reported down

With the conductor down, every compute is `down` for Nova. Deleting an instance then is a *local delete*: Nova
removes the record, its ports, attachments and allocations, but the guest keeps running on its compute until
`nova-compute` finds it and applies `[DEFAULT] running_deleted_instance_action` (`reap` by default). With the
conductor down, the computes cannot run that check either; it runs again at the next `nova-compute` start and every
`running_deleted_instance_poll_interval` (1800 s) after. Avoid deletes until the conductor is back and the computes
are `up`.

## Possible causes

| What you see | Cause |
|---|---|
| Containers stopped or restarting; their log shows the start-up error | Cause 1: the service itself (configuration, database unreachable at start-up) |
| Containers running, queues without consumers, `MessagingTimeout` or connection errors in their log | Cause 2: RabbitMQ (see [RPC timeouts caused by RabbitMQ](rabbitmq-partition-or-alarm-rpc-timeouts.md)) |
| Containers running and consuming, but every request is slow | Cause 3: a dependency they wait on (database, Placement for the scheduler): see [VM creation fails platform-wide](../diagnostics/vm-creation-fails-platform-wide-check-shared-dependencies.md) |

## Resolution

### For cause 1: start the service again

Read the last lines of its log first: a container that exits at start-up will exit again.

> ⚠️ **State-changing.** Restarts the conductor or scheduler on one controller. Requests that this copy was
> processing fail and are retried or buried; queued requests are then consumed. Precondition: the cause of the exit
> is fixed. Undo: none needed.

```bash
sudo systemctl restart kolla-nova_conductor-container.service   # or kolla-nova_scheduler-container.service
```

If the configuration is at fault, fix it on the deploy host and run `kolla-ansible reconfigure -i <inventory> -t
nova`.

### For cause 2: fix RabbitMQ first

Restarting Nova services while RabbitMQ is partitioned or blocked does not help: they reconnect and wait again.

### After the service is back

- Builds queued while the conductor was down finish by themselves; check that none stays in `BUILD` after a few
  minutes.
- Builds that timed out while the scheduler was down are in `ERROR` in cell0. They are not retried: their owners
  build them again, and the failed records can be deleted. `--status ERROR` lists `ERROR` instances of every cell,
  including real instances on computes with disks and volumes: select only the ones of the outage.

```bash
openstack server list --all-projects --status ERROR --long -c ID -c Name -c Host -c "Project ID"
openstack server show <server-id> -c OS-EXT-SRV-ATTR:host -c fault -c created
```

> 🛑 **Destructive.** Deletes the instance record and its fault, which is the only trace of why it failed. Who is
> affected: the project that owns it, which must build again. Precondition: `OS-EXT-SRV-ATTR:host` is empty, the
> fault is `MessagingTimeout` or `No valid host` and `created` falls in the outage window, and you have recorded
> the ID, project and fault. Undo: none.

```bash
openstack server delete <server-id>
```

### A task state left set

A start, stop or reboot sent while the conductor was down can leave the instance with a `task_state` (`powering-off`,
`rebooting`...) that nothing clears. Once the conductor is back, check what the guest really does:

```bash
openstack server show <server-id> -c status -c OS-EXT-STS:power_state -c OS-EXT-STS:task_state
```

Choose the state from the power state: `active` if the guest runs (`Running`), otherwise `error`, then hard-reboot
it (a plain start is refused from `error`: `start` accepts only a stopped instance). When a stop timed out, the
compute has often powered the guest off already: setting `active` would leave a record that says it runs. Resetting
the state is an admin action (`os_compute_api:os-admin-actions:reset_state` is admin-only by default).
A hard reboot is refused while the `task_state` is set (`409 ... Cannot 'reboot' instance <server-id> while it is in
task_state powering-on`): reset the state first. `openstack server set --state` asks for confirmation
(python-openstackclient 10.5.0); `--auto-approve` skips the question, and without a terminal the command fails.

> ⚠️ **State-changing.** Changes the instance record only (its `vm_state`, and clears `task_state`); nothing happens
> to the guest. Precondition: the conductor is back and the power state has been checked as above. Undo: set the
> other state with the same command; the old `task_state` cannot be restored, which is the point.

```bash
openstack server set --auto-approve --state active <server-id>     # the guest runs
openstack server set --auto-approve --state error <server-id>      # the guest is off
```

If the guest is off and was set to `error`, bring it back with a hard reboot:

> ⚠️ **State-changing.** Powers the guest on and recreates its libvirt domain from Nova's records. Precondition: the
> power state shows the guest off (`Shutdown` or `NOSTATE`), and its owner expects it to run. Undo:
> `openstack server stop <server-id>`.

```bash
openstack server reboot --hard <server-id>
```

## Verification

> ⚠️ **State-changing.** The last two commands create and delete a test instance. Needs quota, an image, a flavor
> and a network.

```bash
sudo docker exec rabbitmq rabbitmqctl list_queues -p <vhost> name messages consumers | grep -E '^(conductor|scheduler)\s'
# expected: consumers > 0, messages near 0
openstack compute service list
# expected: every nova-conductor, nova-scheduler and nova-compute up
openstack server create --image <image> --flavor <flavor> --network <network> --wait <test-name>
openstack server delete --wait <test-name>
```

## Validity and limits

- Written from Nova 2026.1: `nova/conductor/api.py` (`wait_until_ready`), `nova/conductor/manager.py`
  (`schedule_and_build_instances`, `_bury_in_cell0`), `nova/compute/utils.py` (`exception_to_dict`),
  `nova/scheduler/rpcapi.py` (`select_destinations` with
  `long_rpc_timeout` and `call_monitor_timeout`), `nova/servicegroup/drivers/db.py`, `nova/conf/rpc.py`,
  `nova/conf/api.py`
  (`list_records_by_skipping_down_cells`), `nova/compute/api.py` (local delete), and Kolla-Ansible 2026.1 (`nova`
  and `nova-cell` role defaults: `nova_scheduler`, `nova_conductor`, `nova_super_conductor` only with
  `enable_cells`). The MessagingTimeout text is oslo.messaging's.
- Lab observations (Kolla-Ansible 22.2.0, 2026.1, 3 controllers + 2 computes, RabbitMQ quorum queues, defaults
  `rpc_response_timeout` 60 and `service_down_time` 60, 2026-10-27): with every `nova_conductor` stopped, a new build
  stayed `BUILD`/`scheduling`, both computes were `down` 51-58 s after the last conductor stopped, the `conductor`
  queue had 0 consumers and 3, then 6 messages; a `server start` of a stopped instance was accepted, `nova-compute`
  logged `MessagingTimeout` 120-122 s later (two runs) and the instance kept `task_state` `powering-on` after the
  conductors were back; `systemctl restart` of the conductors brought the computes `up` in 11 s and the queued build
  `ACTIVE` in 32 s, not resubmitted; `--auto-approve --state error` then `reboot --hard` brought the instance
  `ACTIVE`. With every `nova_scheduler` stopped, a build went `ERROR` 61-63 s after creation (two runs), no host,
  fault `MessagingTimeout`; the cleanup commands ran as written. `Timed out waiting for nova-conductor` was seen
  when the computes started before the conductors at lab start-up, not injected.
- Not run: resize, migrations, evacuate, a stop or reboot of a running instance, a delete while the computes were
  `down` (local delete), the cell0 database unreachable, `--state active`, `kolla-ansible reconfigure`.
- The impact table follows the code paths; actions not listed (console, attach) behave like the closest one but were
  not traced.
- Whether a queued cast survives depends on the queue type and RabbitMQ staying up: with Kolla-Ansible 2026.1's
  quorum queues it does; a RabbitMQ reset drops it.
- Not covered: multi-cell deployments (a super conductor and cell conductors), and `nova-api` or `nova-metadata` down.

## Related

- [RPC timeouts caused by RabbitMQ](rabbitmq-partition-or-alarm-rpc-timeouts.md)
- [Control plane down: what still works for running
  instances](../diagnostics/control-plane-down-what-still-works-for-instances.md)
- [No valid host was found, although capacity exists](../scheduling/no-valid-host-with-free-capacity.md): the other
  common way into cell0.

---

*Provenance: written by agent:ozarland-author, from the Nova and Kolla-Ansible 2026.1 documentation and source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=recovery/nova-conductor-or-scheduler-down-what-still-works.md&title=Feedback%3A+recovery/nova-conductor-or-scheduler-down-what-still-works.md):
your release, your deployment method, and what happened.
