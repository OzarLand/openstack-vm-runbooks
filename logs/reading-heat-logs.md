---
title: Reading the Heat logs. From a failed stack to the service that really failed
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [heat, nova, neutron, cinder]
deployment_methods: [kolla-ansible, any]
environment:
  topology: multinode
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, all-in-one, convergence engine", date: 2026-10-27, note: "PR lab test (No valid host stack), stack commands and greps, a stack timeout, Nova request-id check; Timed out waiting for operation to start, Unexpected exception, Failed to retrieve user_creds, client-error tracebacks (Neutron, Cinder, Keystone) and the legacy engine not run" }
upstream_refs:
  - https://opendev.org/openstack/heat/src/branch/stable/2026.1/heat/engine/stack.py
  - https://opendev.org/openstack/heat/src/branch/stable/2026.1/heat/engine/resource.py
  - https://opendev.org/openstack/heat/src/branch/stable/2026.1/heat/engine/service.py
  - https://opendev.org/openstack/heat/src/branch/stable/2026.1/heat/common/exception.py
  - https://opendev.org/openstack/heat/src/branch/stable/2026.1/heat/common/config.py
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Reading the Heat logs. From a failed stack to the service that really failed

Heat rarely fails by itself: a stack fails because one of its resources failed in Nova, Neutron, Cinder or another
service. The Heat log, and the stack's own events, name the resource and repeat the other service's error; the cause
is then in that service's log. Messages are quoted from Heat 2026.1.

## Where the logs are

**Kolla-Ansible:** `/var/log/kolla/heat/` (`log_dir` and `log_file = $log_dir/<service>.log` in `heat.conf.j2`):
`heat-api.log`, `heat-api-cfn.log` and `heat-engine.log`, from the containers `heat_api`, `heat_api_cfn` and
`heat_engine`. The APIs run under uWSGI, and their request lines are in `heat-api-uwsgi.log` and
`heat-api-cfn-uwsgi.log` (`<service>-uwsgi.log` in the `service-uwsgi-config` role). The engine does the work; read
`heat-engine.log` first.

**Any:** `/var/log/heat/` or the journal.

Before the log, the stack itself says which resource failed and why:

```bash
openstack stack show <stack> -c stack_status -c stack_status_reason
openstack stack resource list <stack> --filter status=FAILED -n 5     # -n: into nested stacks
openstack stack event list <stack> --nested-depth 5 | grep FAILED
```

## The status reason

When a resource fails, Heat records a `ResourceFailure` whose text is `<error type>: <path>: <message>`
(`HeatExceptionWithPath` in `common/exception.py`), for example `ResourceInError: resources.<name>: Went to status
ERROR due to "<reason>"`. `ResourceInError` (`Went to status <status> due to "<reason>"`) means the other service
put the resource in an error state, and `<reason>` is that service's own fault text: for a server, Nova's fault
as `Message: <fault message>, Code: <code>`. Captured on a 2026.1 lab (server on a flavor no host fits):

```text
Resource CREATE failed: ResourceInError: resources.web: Went to status ERROR due to "Message: No valid host was found. , Code: 500"
```

Read `<reason>` as a message from the other service, and continue in that service's log.

Heat retries a failed resource create (`action_retry_limit`, default 5) before failing the stack: on the lab, 6
attempts in 100 s, each with a new server (only the last one stays, in `ERROR`), and one `CREATE_FAILED` event per
attempt in `openstack stack event list`.

## In heat-engine.log

| Line (message text) | Level | What it means |
|---|---|---|
| `Stack <action> <status> (<name>): <reason>` | INFO | A stack changed state, for example `Stack CREATE FAILED (<name>): Resource CREATE failed: <error>` (the reason is built as `'Resource %s failed: %s'` in `stack.py`). The reason is the same as `stack_status_reason`. |
| `<ACTION>: <Type> "<name>" [<physical id>] Stack "<stack>" [<stack id>]` with a traceback (`<Type>` is the Python class name, for example `Server`, not `OS::Nova::Server`) | INFO | A resource action failed. When the other service refused the call, the traceback's last lines are its client error (a `novaclient`, `neutronclient`, `cinderclient` or `keystoneauth` exception, with the HTTP status and message). When the resource went to an error state, the traceback ends with Heat's own `heat.common.exception.ResourceInError: Went to status ERROR due to "<reason>"` (seen on the lab). The `[<physical id>]` is the id in the other service: grep it there; with retries, each attempt logs a new one. |
| `Timed out waiting for operation to start` | WARNING | The engine that received a request did not start it within `rpc_response_timeout`: engines overloaded or RabbitMQ slow. Logged only by `NotifyEvent.wait()` in `service.py`: suspend, resume, check and restore, and the legacy engine's update and delete. With `convergence_engine` (the default), create, update and delete never log it. |
| `Unexpected exception in <function>: <message>` | ERROR | An engine thread failed outside a resource; the stack may stay `IN_PROGRESS`. |
| `Failed to retrieve user_creds` with a traceback | ERROR | During a stack delete, the credentials stored for the stack could not be read (truncated when saved, or `auth_encryption_key` changed): `_try_get_user_creds()` in `stack.py`. The error is ignored on purpose and **the delete proceeds**; only the clean-up of the stack's Keystone trust is skipped, so the trust may be left behind for manual clean-up (`openstack trust list`). It is not the cause of a `DELETE_FAILED` stack. |

Sources: `heat/engine/stack.py`, `heat/engine/resource.py`, `heat/engine/service.py`.

A stack that stays `CREATE_IN_PROGRESS` until it fails with a timeout hit the stack timeout (`stack_action_timeout`,
default 3600 s, or the `--timeout` given at create). With the convergence engine (the default), the lab showed the
reason `Timed out`, and the resource that was still running went to `CREATE_FAILED` with `CREATE aborted (Task create
from <Type> "<name>" Stack "<stack>" [<stack id>] Timed out)`; look for that resource and read the other service's log
for it. In `heat-engine.log` (captured on the lab, `OS::Heat::TestResource`, `--timeout 1`):

```text
INFO heat.engine.scheduler [...] Task create from TestResource "slow" Stack "lab-stack-slow" [<stack-id>] timed out
INFO heat.engine.stack [...] Stack CREATE FAILED (lab-stack-slow): Timed out
```

The same timeout bounds a delete: a delete that takes longer ends `DELETE_FAILED`, `Timed out`. The reason `Create
timed out` (`'%s timed out' % action.title()` in `stack.py`) exists in the source but was not seen on the lab.

```bash
# Kolla-Ansible, on the controllers
grep -n "Stack .* FAILED" /var/log/kolla/heat/heat-engine.log | tail
grep -n "<physical-id>" /var/log/kolla/heat/heat-engine.log
```

## Following the failure into the other service

| Resource type | Physical id is | Continue with |
|---|---|---|
| `OS::Nova::Server` | the server id | `openstack server show <id> -c fault`, then [the Nova logs](reading-nova-logs.md) |
| `OS::Neutron::Port`, `OS::Neutron::Net`, … | the Neutron resource id | `neutron-server.log` for that id ([the Neutron logs](reading-neutron-logs-ml2-ovs.md)) |
| `OS::Cinder::Volume` | the volume id | `openstack volume show <id>`, then [the Cinder logs](reading-cinder-logs.md) |

Heat calls the other services with its own request context. On the 2026.1 lab, Heat's request id did not appear in
Nova's logs: `nova-api` logged Heat's server create and delete with `[None req-<nova-request-id> …]` (no global request
id), even when Heat's own context carried one. Link the two by the physical id and the time.

## Validity and limits

- Every message is quoted from Heat 2026.1. Captured on a lab (Kolla-Ansible 22.2.0, 2026.1, all-in-one, convergence
  engine, 2026-10-27): the stack commands, the status reason with Nova's `No valid host was found.`, the
  `CREATE: Server …` line and its traceback, `Stack CREATE FAILED`, the 6 create attempts, the stack timeout
  (`Timed out`), and the request id check (Nova calls only). Not seen on the lab: `Timed out waiting for operation to
  start`, `Unexpected exception in …`, `Failed to retrieve user_creds`, `Create timed out`, a client error in a
  traceback (Neutron, Cinder, Keystone), the legacy engine.
- Software deployments, signals and the CFN API are not detailed.

## Related

- [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md)
- [Reading the Nova logs](reading-nova-logs.md), [the Neutron logs](reading-neutron-logs-ml2-ovs.md), [the Cinder logs](reading-cinder-logs.md): where a failed resource's error continues
- [Logs index](README.md)

---

*Provenance: written by agent:ozarland-author, from Heat 2026.1 source code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Freading-heat-logs.md&title=Feedback%3A+logs%2Freading-heat-logs.md):
your release, your deployment method, and what happened.
