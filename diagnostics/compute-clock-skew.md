---
title: The logs of one compute do not line up with the controllers. Its clock has drifted
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [nova, neutron, ntp]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers, 2 computes
  network: ml2/ovs (VXLAN)
  storage: local ephemeral disk on the computes, no Cinder
  os: ubuntu-24.04 (noble), docker, systemd-timesyncd
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/install-guide/environment-ntp.html
  - https://docs.openstack.org/nova/latest/admin/service-groups.html
authored_by: agent:ozarland-author
last_verified: 2026-10-21
---

# The logs of one compute do not line up with the controllers. Its clock has drifted

A compute whose clock is minutes off keeps working. Nothing fails and nothing alerts. The problem shows during an
investigation: the timeline you build from several hosts is wrong.

## Symptom

- While following one request or one incident across hosts, the log lines of one compute appear **minutes before**
  the events that caused them on the controllers (or minutes after).
- Everything else looks healthy: `nova-compute` of that compute is `up`, its agents are alive, new instances boot.
- On the compute, `timedatectl` shows the clock is not synchronised:

```text
$ timedatectl
...
System clock synchronized: no
              NTP service: inactive
...
```

(or `NTP service: active` with a time source that cannot be reached).

## Impact

- **No outage** for a drift of a few minutes on a compute: on the lab, 300 s back changed nothing visible for Nova,
  Neutron or the instances.
- **Misleading investigations:** the logs of the compute are stamped with its own wrong clock. A timeline built
  across hosts puts events in the wrong order.
- **Not covered here and likely more serious:** a drift on a controller (Keystone token issue and expiry, Galera,
  RabbitMQ), a forward drift, a much larger offset, TLS certificate validity checks. Do not assume this page applies
  to them.

## Quick checks

Run on each host (controllers and computes):

```bash
# 1. Is the clock synchronised, and is a time service running?
timedatectl

# 2. Details of the synchronisation (systemd-timesyncd)
timedatectl timesync-status
#    with chrony instead:
chronyc tracking
```

Compare the clocks of all hosts in one run, from the deployment host:

**Kolla-Ansible:**

```bash
ansible -i <inventory> all -m command -a 'date -u +%s'
```

**Any:**

```bash
for h in controller-1 compute-1 compute-2; do echo "$h $(ssh $h date -u +%s)"; done
```

On a controller, check that the services of the compute are healthy anyway (this is what makes the drift silent):

```bash
openstack compute service list --service nova-compute --host <compute>
openstack network agent list --host <compute>
```

A difference of more than a few seconds on one host, with `System clock synchronized: no` there, is this problem.

## Diagnosis

### Time synchronisation on the host

Find why the host lost its time:

```bash
systemctl status systemd-timesyncd          # or: systemctl status chrony
journalctl -u systemd-timesyncd --since "-1d"
timedatectl show -p NTPSynchronized --value  # yes / no
```

Look for: the service stopped or disabled, no server configured, the server unreachable (firewall, routing), or a
clock set by hand (`date -s`, a hardware clock reset after a reboot).

### Nova and Neutron

Why they do not complain: the liveness of `nova-compute` is recorded by `nova-conductor` (the `updated_at` of the
service record, with the database driver of the service group) and that of the Neutron agents by `neutron-server`.
Both run on the controllers, with the controllers' clocks. The expiry of Keystone tokens is checked by the services
that receive them, not by the compute. What the compute stamps itself is its logs.

To measure the offset in the logs, compare the same event on both sides, for example a request id:

```bash
# On a controller
grep <request-id> /var/log/kolla/nova/nova-conductor.log | head -n 3
# On the compute
grep <request-id> /var/log/kolla/nova/nova-compute.log | head -n 3
```

## Possible causes

### Cause 1: the time service is stopped

`NTP service: inactive` in `timedatectl`. The host drifts from the moment it stopped. This is the case reproduced on
the lab (with a clock also set back by hand).

### Cause 2: the time source cannot be reached

The service runs, but `timedatectl timesync-status` (or `chronyc sources`) shows no usable server. Not reproduced on
the lab.

### Cause 3: the clock was set by hand

Someone ran `date -s`, or the hardware clock was wrong after a reboot, and synchronisation is off. Part of the lab
reproduction.

## Resolution

### For cause 1 and cause 3

On the compute, start the time service. With a large offset, systemd-timesyncd steps the clock to the right time at
its first synchronisation.

> ⚠️ **State-changing.** Starts time synchronisation on this host; the clock jumps by the offset (on the lab, 300 s
> forward). Affects this host only: its log timestamps jump, and timers of its services see the jump. Precondition:
> a reachable time source. Undo: stop the service (the clock then drifts again).

```bash
systemctl start systemd-timesyncd
systemctl enable systemd-timesyncd
```

With chrony instead:

```bash
systemctl start chrony
```

Wait for the synchronisation:

```bash
timedatectl show -p NTPSynchronized --value   # repeat until it prints "yes"
```

Then, for the logs written during the drift, correct the timestamps of that host by the measured offset when you
build the timeline.

### For cause 2

Make the time source reachable (firewall, routing, the server configured in `/etc/systemd/timesyncd.conf` or the
chrony configuration), then restart the service.

> ⚠️ **State-changing.** Restarts time synchronisation on this host; the clock jumps by the offset once a source is
> reached. Affects this host only. Undo: none needed.

```bash
systemctl restart systemd-timesyncd
```

Not tested on the lab.

## Verification

On the compute:

```bash
timedatectl          # System clock synchronized: yes, NTP service: active
```

From the deployment host:

```bash
ansible -i <inventory> all -m command -a 'date -u +%s'
```

Expected: the clock of the compute within a few seconds of the controllers (the lab checked at most 5 s from the
first controller), and its services still healthy:

```bash
openstack compute service list --service nova-compute --host <compute>
openstack network agent list --host <compute>
```

To avoid finding this during the next incident, monitor the synchronisation state of every host
(`NTPSynchronized`, or chrony's offset) rather than waiting for OpenStack to complain.

## Lab observations

On the lab, the time service of one compute was stopped and its clock set 300 s back; an instance was then created
on that compute. Observed for 180 s:

| | Before | During (180 s) | After (start + 60 s) |
|---|---|---|---|
| Clock of compute-2 against the other hosts | equal | **301 s behind** | equal |
| Nova and Neutron services | 22 ok | **22 ok** (`nova-compute` of compute-2 up, its Open vSwitch agent alive) | 22 ok |
| New instance on compute-2 | – | **ACTIVE in 13 s, no fault** | deleted |
| Requests every 0.5 s to instances on compute-1 and compute-2 | – | 381 and 382 requests, 0 lost | 565 requests, 0 lost |
| WARNING/ERROR lines of `nova-compute` and the Open vSwitch agent on compute-2 | usual noise only | usual noise only, **nothing about time** | usual noise only |

- During the drift, a line written by compute-2 at 14:05 real time was stamped 14:00:22.
- Starting systemd-timesyncd brought the clock back within 5 s of the first controller.
- The lab expected the heartbeats of the compute to fail. None did, and that is the finding: the drift is silent.

## Validity and limits

- Observed on one lab build with the configuration above: OpenStack 2026.1, Kolla-Ansible 22.2.0, ML2/OVS, Docker,
  Ubuntu 24.04 with systemd-timesyncd, Nova's database service group driver.
- Observed: a 300 s backward drift on one compute, for 180 s; services up, new instance created, no loss, no time
  warning in the logs; recovery by starting systemd-timesyncd.
- Not run: chrony, cause 2 (unreachable source), the log comparison by request id, `enable` of the service.
- Not tested: a drift on a controller, a forward drift, an offset larger than 300 s or lasting longer than a few
  minutes, TLS certificate checks, other service group drivers, OVN, other releases, other deployment tools,
  production scale. Any of these may break things this page says stay healthy.

## Related

- [What a VM creation traverses, and tracing by request id](what-a-vm-creation-traverses-and-tracing-by-request-id.md):
  the timeline work this drift corrupts.
- [Expired TLS certificates or inconsistent Fernet keys](expired-certificates-and-fernet-keys-break-every-api-call.md):
  where time does break things.
- [nova-compute down, instances still running](../recovery/nova-compute-down-instances-still-running.md)
- OpenStack install guide, Network Time Protocol: https://docs.openstack.org/install-guide/environment-ntp.html
- Nova, service groups: https://docs.openstack.org/nova/latest/admin/service-groups.html
- Source: lab reproduction by @soufian-zaouam (OpenStack 2026.1, Kolla-Ansible 22.2.0).

---

*Provenance: written by agent:ozarland-author, from a lab reproduction by @soufian-zaouam on 2026.1.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*
