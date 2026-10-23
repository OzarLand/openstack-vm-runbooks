---
title: nova-compute is down and the compute refuses an SSH login, but the host is alive
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [nova, libvirt, ssh]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers, 2 computes
  network: ml2/ovs (VXLAN)
  storage: local instance disks on the computes (no Cinder)
  os: ubuntu-24.04 (noble), docker
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: kolla-ansible all-in-one, date: 2026-10-23, note: "full procedure on a disposable lab; DenyUsers cause and fix, plus the journal messages of the AllowUsers and missing-account causes; ping and nc checks run" }
upstream_refs:
  - https://docs.openstack.org/nova/latest/admin/evacuate.html
  - https://docs.openstack.org/nova/latest/admin/node-down.html
  - https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/compute-service.html
last_verified: 2026-10-23
---

# nova-compute is down and the compute refuses an SSH login, but the host is alive

Two independent faults on one host: its `nova-compute` agent stopped, and the SSH user of your monitoring or
automation is refused there. Together they look like a dead host: Nova says the compute is down, and your tools
cannot read it. It is not dead. An SSH authentication failure says nothing about whether a host is alive; only a
failure at the TCP level does. Evacuating it would start a second copy of each instance.

## Symptom

```text
$ openstack compute service list --service nova-compute -c Host -c State
| compute-1 | up   |
| compute-2 | down |

$ ssh -o BatchMode=yes <monitoring-user>@compute-2 true
<monitoring-user>@compute-2: Permission denied (publickey).
```

The same user and key log in to every other host. The host answers on TCP port 22, other users can log in, and its
instances keep running and stay `ACTIVE` in Nova.

## Impact

- Running instances on the host: none, as long as nobody acts on the wrong diagnosis.
- Nova cannot act on the host's instances while the agent is down.
- Your monitoring or automation reads nothing on that host. If it treats "cannot log in" as "host down", it reaches
  the wrong conclusion, and an automated evacuation would run each instance twice on the same disks.

## Quick checks

From a controller or the host your monitoring runs on:

```bash
# 1. Does the host answer on the network and on TCP 22?
ping -c 3 <compute-host>
nc -zv -w 5 <compute-host> 22

# 2. Where does the SSH connection fail? A refusal after authentication means the host is alive
ssh -v -o BatchMode=yes -o ConnectTimeout=5 <monitoring-user>@<compute-host> true 2>&1 | tail -n 5
```

How to read step 2:

| Last lines | Meaning |
|---|---|
| `Connection timed out`, `No route to host` | Not reached at TCP level: the host may be down or cut off. See [fence before you evacuate](compute-host-failure-fence-before-evacuate.md) |
| `Connection refused` | Host up, nothing listening on 22 (sshd stopped) |
| `Permission denied (publickey)` after the key exchange | Host up, sshd up, this user refused: this page |

On a controller, with admin credentials:

```bash
# 3. The compute service and the instances Nova places on the host
openstack compute service list --service nova-compute --host <compute-host> --long
openstack server list --all-projects --host <compute-host> --long -c ID -c Status -c "Power State"
```

## Diagnosis

Log in to the compute as a user that is accepted (an administrator account), or use its console.

### Nova agent and instances

**Kolla-Ansible:**

```bash
systemctl is-active kolla-nova_compute-container.service    # inactive or failed: both mean stopped
docker ps -a --filter name='^nova_compute$' --filter name='^nova_libvirt$' --format '{{.Names}}\t{{.Status}}'
docker exec nova_libvirt virsh list --all
```

**Any:**

```bash
systemctl is-active nova-compute libvirtd
virsh list --all
```

If `nova_compute` is stopped, `nova_libvirt` runs and the instances are `running`, the Nova side is
[nova-compute down, instances still running](nova-compute-down-instances-still-running.md).

### SSH refusal

```bash
# The effective sshd rules for users
sudo sshd -T | grep -i -e '^denyusers' -e '^allowusers' -e '^denygroups' -e '^allowgroups'
ls -la /etc/ssh/sshd_config.d/

# Why sshd refused, in its journal (Ubuntu 24.04 logs as sshd; newer OpenSSH also as sshd-session)
sudo journalctl --no-pager --since -15min -t sshd -t sshd-session | grep -i '<monitoring-user>' | tail -n 10

# The account and its key
getent passwd <monitoring-user>
sudo chage -l <monitoring-user>
sudo ls -la ~<monitoring-user>/.ssh/
```

Your own failed attempts land in that journal too. Read it right after a fresh attempt, and keep in mind that a
window of 15 minutes still shows refusals from before the fix.

## Possible causes

The two faults are independent; handle both.

| Cause | How to tell |
|---|---|
| `nova-compute` stopped (crash, memory, failed restart) | See [nova-compute down, instances still running](nova-compute-down-instances-still-running.md) |
| A `DenyUsers` rule names the user | `sshd -T` shows it; journal: `User <monitoring-user> from <address> not allowed because listed in DenyUsers` |
| An `AllowUsers` or `AllowGroups` rule does not include the user | `sshd -T` shows an allow list without the user; the journal says it is not listed |
| The account does not exist on that host | `getent passwd` returns nothing; journal: `Invalid user <monitoring-user>` |
| The key is missing or changed in `authorized_keys`, or the account is expired or locked | `sshd -T` has no rule against the user; check `authorized_keys` and `chage -l` |

The client sees `Permission denied (publickey)` in every one of these SSH cases; only the server's journal tells
them apart.

## Resolution

### Step 1: restore the access of the monitoring user

Keep your administrator session open while you change sshd, and test from a second session before closing it.

For a rule in a drop-in or in `/etc/ssh/sshd_config`: remove or correct the line that refuses the user, then check
and reload.

> ⚠️ **State-changing.** Changes who may log in to this host over SSH. A reload keeps open sessions; a configuration
> error could lock out new logins, which is why `sshd -t` runs first. Undo: put the previous line back and reload.

On the compute:

```bash
sudoedit /etc/ssh/sshd_config.d/<drop-in>.conf     # or /etc/ssh/sshd_config
sudo sshd -t && sudo systemctl reload ssh.service
sudo sshd -T | grep -i -e '^denyusers' -e '^allowusers'
```

For a missing or changed key: put the monitoring user's public key back in its `authorized_keys`.

> ⚠️ **State-changing.** Grants this key access to the host as that user. Precondition: the key is the one your
> monitoring uses, from a trusted source. Undo: remove the line.

```bash
sudoedit ~<monitoring-user>/.ssh/authorized_keys
```

Then read the host again with the monitoring user before deciding anything about it.

### Step 2: start nova-compute again

> ⚠️ **State-changing.** Starts `nova-compute` on this compute. Running instances are not touched; the agent
> resynchronises with Nova on start.

**Kolla-Ansible:**

```bash
sudo systemctl start kolla-nova_compute-container.service
```

**Any:**

```bash
sudo systemctl start nova-compute
```

Then find why it stopped, as in
[nova-compute down, instances still running](nova-compute-down-instances-still-running.md).

### Never evacuate on the strength of a refused login

An evacuation needs the host fenced first: powered off or isolated through a channel that does not depend on it.
See [Compute host failure: fence before you evacuate](compute-host-failure-fence-before-evacuate.md).

## Verification

```bash
# From the monitoring host: the user logs in to this compute as to the others
ssh -o BatchMode=yes -o ConnectTimeout=5 <monitoring-user>@<compute-host> id -un

# On a controller: the compute service is up
openstack compute service list --service nova-compute --host <compute-host>
```

Expected: the user name printed with exit code 0, and the service `up`. On the compute,
`sudo sshd -T | grep -i '^denyusers'` prints nothing (or no longer names the user).

## Lab observations

On the lab, `nova_compute` was stopped on compute-2 and sshd there was given a `DenyUsers` rule for one dedicated
monitoring user only; the user running the procedure kept its access.

| | Before | During (90 s) | After the fix |
|---|---|---|---|
| `nova-compute` of compute-2 | – | **`enabled down`** | `enabled up` |
| Instance on compute-2 | running | **`ACTIVE`, power state 1 (Running) in Nova; `running` in `virsh`** | – |
| `kolla-nova_compute-container.service` on compute-2 | active | **failed** | active |
| `ssh.service` on compute-2 | active | active | – |
| `denyusers` in `sshd -T` on compute-2 | none | **the monitoring user**, from a drop-in | none |
| Login as the monitoring user from compute-1 to compute-2 | – | **exit code 255, `Permission denied (publickey).`** | – |
| sshd journal of compute-2, last 15 minutes | no refusal | **`User <monitoring-user> from <address> not allowed because listed in DenyUsers`** | still shows the refusals made during the failure |

- During the failure, the monitoring user logged in to the three controllers and to compute-1, and every command
  there succeeded; on compute-2 every attempt failed with `Permission denied (publickey).`
- After the fix (drop-in removed, `sshd -t`, reload of `ssh.service`, `nova_compute` started through its unit), the
  user was accepted on all five hosts and the compute service was `up`.
- The systemd unit of the stopped `nova_compute` was `failed`, not `inactive`. Accept both when reading it.
- Whether compute-2 answered to ping was not recorded.
- One run, 2026-10-14.

## Validity and limits

- Observed once, on one lab build with the configuration above: OpenStack 2026.1, Kolla-Ansible 22.2.0, ml2/OVS,
  Docker, Ubuntu 24.04 (OpenSSH as shipped), 3 controllers and 2 computes.
- Not tested: OVN, other releases, other deployment tools, production scale, other distributions or OpenSSH
  versions (journal tags and messages may differ).
- Observed: the `DenyUsers` cause and its fix (remove the rule, check, reload), and starting `nova_compute` again.
- Not observed: the other SSH causes in the table (`AllowUsers`, missing account, key or expired account), the ping
  check, the `nc` check, and a host with `nova_compute` stopped and no instance.
- The `ssh -v` reading table is standard OpenSSH client behaviour; the lab recorded only the
  `Permission denied (publickey)` case.

## Related

- [nova-compute down, instances still running](nova-compute-down-instances-still-running.md): the Nova half of
  this failure.
- [Compute host failure: fence before you evacuate](compute-host-failure-fence-before-evacuate.md): what to do when
  the host really is gone.
- [Control plane down: what still works for instances](../diagnostics/control-plane-down-what-still-works-for-instances.md)
- Nova: [Evacuate instances](https://docs.openstack.org/nova/latest/admin/evacuate.html).
- OpenSSH: `sshd_config(5)` (`DenyUsers`, `AllowUsers`, `Include`).
- Source: lab reproduction by the OzarLand maintainer (OpenStack 2026.1, Kolla-Ansible 22.2.0).
