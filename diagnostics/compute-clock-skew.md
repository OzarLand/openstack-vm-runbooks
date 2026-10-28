---
title: The logs of one compute do not line up with the controllers. Its clock has drifted
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, neutron, ntp]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers, 2 computes
  network: ml2/ovs (VXLAN)
  storage: local ephemeral disk on the computes, no Cinder
  os: ubuntu-24.04 (noble), docker, systemd-timesyncd
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, multinode (3 controllers, 2 computes)", date: 2026-10-27, note: "compute clock 300 s back with timesyncd stopped and with it disabled, and cause 2 (unreachable NTP server), on a disposable lab; quick checks (timesync-status started the stopped service), request-id comparison, fixes with sudo and Verification run; chrony and the plain ssh loop not run" }
upstream_refs:
  - https://docs.openstack.org/install-guide/environment-ntp.html
  - https://docs.openstack.org/nova/latest/admin/service-groups.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
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
```

> ⚠️ **State-changing when systemd-timesyncd is only stopped.** Check 2 then starts systemd-timesyncd over D-Bus: it
> steps the clock, and the offset can no longer be measured. Precondition: measure the offset first with the `date`
> comparison below, and skip check 2 when `timedatectl` shows `NTP service: inactive`. Undo: `sudo systemctl stop
> systemd-timesyncd` stops the service, but does not undo the step.

```bash
# 2. Details of the synchronisation (systemd-timesyncd). Not read-only if the service is only stopped: see below.
timedatectl timesync-status
#    with chrony instead:
chronyc tracking
```

`timedatectl timesync-status` asks systemd-timesyncd over D-Bus, and D-Bus starts the service if it is stopped but
not disabled (`systemctl stop`): on the lab, the clock jumped back to the right time at once and the offset was
gone before it was measured. When `timedatectl` shows `NTP service: inactive`, measure the offset first (below) and
skip check 2. After `timedatectl set-ntp false` (service disabled), it only prints `Failed to query server: Unit
dbus-org.freedesktop.timesync1.service not found.`

Compare the clocks of all hosts in one run, from the deployment host:

**Kolla-Ansible:**

```bash
ansible -i <inventory> all -m command -a 'date -u +%s'
```

**Any:**

```bash
for h in controller-1 compute-1 compute-2; do echo "$h $(ssh $h date -u +%s)"; done
```

(Needs SSH from where you run it to each host with known host keys; on the lab, the Kolla-Ansible deployment host
had neither for plain `ssh`, and the Ansible variant was used.)

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
sudo grep <request-id> /var/log/kolla/nova/nova-conductor.log | head -n 3
# On the compute
sudo grep <request-id> /var/log/kolla/nova/nova-compute.log | head -n 3
```

On the lab, with the compute 300 s behind, the conductor stamped `07:47:18.617` and the compute's `Claim successful`
line, which comes after it, `07:42:19.517`: 299 s apart. Without `sudo`, `grep` gets `Permission denied`.

## Possible causes

### Cause 1: the time service is stopped

`NTP service: inactive` in `timedatectl`. The host drifts from the moment it stopped. This is the case reproduced on
the lab (with a clock also set back by hand).

### Cause 2: the time source cannot be reached

The service runs, but `timedatectl timesync-status` (or `chronyc sources`) shows no usable server. On the lab
(systemd-timesyncd pointed at an address that does not answer): `System clock synchronized: no`, `NTP service:
active`, `Packet count: 0` in `timesync-status`, and `Timed out waiting for reply from <server>:123` in
`journalctl -u systemd-timesyncd`.

### Cause 3: the clock was set by hand

Someone ran `date -s`, or the hardware clock was wrong after a reboot, and synchronisation is off. Part of the lab
reproduction.

## Resolution

### For cause 1 and cause 3

On the compute, start the time service. With a large offset, systemd-timesyncd steps the clock to the right time at
its first synchronisation.

> ⚠️ **State-changing.** Starts time synchronisation on this host; the clock jumps by the offset (on the lab, 300 s
> forward). Affects this host only: its log timestamps jump, and timers of its services see the jump. Precondition:
> a reachable time source. Undo: `sudo systemctl disable --now systemd-timesyncd` (the step enables it across
> reboots; the clock then drifts again).

```bash
sudo systemctl start systemd-timesyncd
sudo systemctl enable systemd-timesyncd
```

With chrony instead:

```bash
sudo systemctl start chrony
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
sudo systemctl restart systemd-timesyncd
```

On the lab, once the source answered again, the clock was synchronised 1 s after the restart.

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

A second run (2026-10-27, three controllers and two computes, same build) set compute-2 300 s back three times:

- With the service stopped by `systemctl stop`, `timedatectl timesync-status` (*Quick checks* 2) started it through
  D-Bus and the clock was right again at once; `timedatectl`, `timedatectl show`, `systemctl status` and
  `journalctl` did not. With `timedatectl set-ntp false`, check 2 failed harmlessly.
- During the drift: Ansible `date -u +%s` showed compute-2 300 s behind the five other hosts; `nova-compute` up and
  the Open vSwitch agent alive; an instance on compute-2 ACTIVE in 27 s; the request id 299 s apart between
  `nova-conductor` and `nova-compute`; no time-related WARNING or ERROR in `nova-compute` or the Open vSwitch agent.
- `sudo systemctl start` and `enable systemd-timesyncd`: synchronised within 1 s. Without `sudo`: `Interactive
  authentication required`.
- Cause 2 (the only server an address that does not answer): reproduced as described above; fixed by restoring the
  server and `sudo systemctl restart systemd-timesyncd`.

## Validity and limits

- Observed on one lab build with the configuration above: OpenStack 2026.1, Kolla-Ansible 22.2.0, ML2/OVS, Docker,
  Ubuntu 24.04 with systemd-timesyncd, Nova's database service group driver.
- Observed: a 300 s backward drift on one compute, for 180 s; services up, new instance created, no loss, no time
  warning in the logs; recovery by starting systemd-timesyncd.
- Not run: chrony; the plain `ssh` comparison loop (Ansible variant used). Cause 2, the log comparison by request
  id and `enable` of the service were run in the second lab run.
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

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=diagnostics/compute-clock-skew.md&title=Feedback%3A+diagnostics/compute-clock-skew.md):
your release, your deployment method, and what happened.
