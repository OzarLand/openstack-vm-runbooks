---
title: Reading the kernel log of a compute (dmesg, journalctl -k). OOM killer, KVM, links and bonds, MTU, disks, conntrack, hung tasks
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [nova, libvirt, qemu, ovs, linux]
deployment_methods: [any]
environment:
  topology: any
  network: any
  storage: any
  os: any linux with systemd
risk: read-only
confirmed_by: []
upstream_refs:
  - https://github.com/systemd/systemd/blob/main/man/journalctl.xml
  - https://github.com/util-linux/util-linux/blob/master/sys-utils/dmesg.1.adoc
  - https://github.com/torvalds/linux/blob/v7.3/mm/oom_kill.c
  - https://github.com/torvalds/linux/blob/v7.3/include/linux/sched.h
  - https://github.com/openvswitch/ovs/blob/v3.7.1/vswitchd/ovs-vswitchd.8.in
  - https://github.com/torvalds/linux/blob/v7.3/kernel/hung_task.c
  - https://github.com/torvalds/linux/blob/v7.3/Documentation/admin-guide/sysctl/kernel.rst
  - https://github.com/torvalds/linux/blob/v7.3/net/netfilter/nf_conntrack_core.c
  - https://github.com/torvalds/linux/blob/v7.3/Documentation/networking/nf_conntrack-sysctl.rst
  - https://github.com/torvalds/linux/blob/v7.3/drivers/net/bonding/bond_main.c
  - https://github.com/torvalds/linux/blob/v7.3/drivers/net/bonding/bond_3ad.c
  - https://github.com/torvalds/linux/blob/v7.3/include/net/bonding.h
  - https://github.com/torvalds/linux/blob/v7.3/drivers/net/ethernet/intel/ixgbe/ixgbe_main.c
  - https://github.com/torvalds/linux/blob/v7.3/drivers/net/ethernet/mellanox/mlx5/core/en_main.c
  - https://github.com/torvalds/linux/blob/v7.3/net/ipv6/addrconf.c
  - https://github.com/torvalds/linux/blob/v7.3/net/openvswitch/vport.c
  - https://github.com/torvalds/linux/blob/v7.3/block/blk-mq.c
  - https://github.com/torvalds/linux/blob/v7.3/block/blk-core.c
  - https://github.com/torvalds/linux/blob/v7.3/fs/ext4/super.c
  - https://github.com/torvalds/linux/blob/v7.3/fs/xfs/xfs_fsops.c
  - https://github.com/torvalds/linux/blob/v7.3/arch/x86/kvm/vmx/vmx.c
  - https://github.com/torvalds/linux/blob/v7.3/arch/x86/kvm/svm/svm.c
  - https://github.com/torvalds/linux/blob/v7.3/arch/x86/kvm/x86.c
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Reading the kernel log of a compute (dmesg, journalctl -k). OOM killer, KVM, links and bonds, MTU, disks, conntrack, hung tasks

Many instance problems that OpenStack only sees as "the instance stopped", "the port is down" or "the volume is
slow" are explained by one line in the kernel log of the compute. The kernel log is not in the OpenStack logs, nor in
Kolla-Ansible's `/var/log/kolla`: read it on the host.

## Reading the kernel log

All commands on this page are read-only. Run them on the compute, as root (reading the kernel ring buffer can be
restricted to root by `kernel.dmesg_restrict`).

### journalctl -k

`journalctl -k` (`--dmesg`) shows only kernel messages; it implies the current boot unless `-b` says otherwise
([`journalctl(1)`](https://github.com/systemd/systemd/blob/main/man/journalctl.xml)):

```bash
journalctl -k                                   # current boot
journalctl -k -b -1                             # previous boot: needs a persistent journal
journalctl --list-boots                         # which boots the journal still holds
journalctl -k --since "2026-10-26 14:00" --until "2026-10-26 14:30"
journalctl -k -p warning                        # warning and more severe only
journalctl -k -o short-iso-precise              # ISO timestamps with microseconds, to line up with other logs
journalctl -k -f                                # follow
```

The previous boot is only there if the journal is persistent (`/var/log/journal` exists, or `Storage=persistent`);
otherwise the kernel lines before a crash or a reboot are lost. Configuring the journal is covered on the systemd and
journald page (planned).

### dmesg

`dmesg` reads the kernel ring buffer directly: it works without journald, but only for the current boot, and the
buffer wraps on a busy host. The util-linux manual warns that `dmesg -T` (human time) can be **inaccurate**: the time
source is not updated after a suspend and resume
([`dmesg(1)`](https://github.com/util-linux/util-linux/blob/master/sys-utils/dmesg.1.adoc)). Prefer the journal's
timestamps when you line kernel lines up with OpenStack logs.

```bash
dmesg -T --level=err,warn | tail -50
dmesg --time-format iso | tail -50
```

## What to look for

All the messages below are format strings from Linux v7.3, not captured lines. Distribution kernels can word them
differently, and driver messages depend on the driver; search for the stable part of the message.

### OOM killer killing QEMU

When the host runs out of memory, the OOM killer picks a process, and QEMU processes are often the largest. From
[`mm/oom_kill.c`](https://github.com/torvalds/linux/blob/v7.3/mm/oom_kill.c):

```text
<command> invoked oom-killer: gfp_mask=<mask>(<flags>), order=<n>, oom_score_adj=<n>
... memory state and the task table ...
Out of memory: Killed process <pid> (<command>) total-vm:<n>kB, anon-rss:<n>kB, file-rss:<n>kB, shmem-rss:<n>kB, UID:<uid> pgtables:<n>kB oom_score_adj:<n>
oom_reaper: reaped process <pid> (<command>), now anon-rss:<n>kB, file-rss:<n>kB, shmem-rss:<n>kB
```

- The first line names the process that **asked** for memory, not the one killed.
- `Out of memory:` is a host-wide OOM; `Memory cgroup out of memory:` means a cgroup limit was hit (for example a
  container's memory limit), not the whole host.
- `<command>` is the task's `comm`, which the kernel cuts to 15 characters (`TASK_COMM_LEN` is 16 with the final
  null byte, in [`include/linux/sched.h`](https://github.com/torvalds/linux/blob/v7.3/include/linux/sched.h)). For an
  instance it is the QEMU binary name, cut: `qemu-kvm`, or `qemu-system-x86` for `qemu-system-x86_64`. The process
  that asked for memory, in the first line, can also be a QEMU vCPU thread, whose name looks like `CPU 0/KVM`. Match the pid with the instance: the instance log in `/var/log/libvirt/qemu/` then ends with
  `shutting down, reason=crashed`, and `nova-compute` logs `Instance shutdown by itself` (see the libvirt and QEMU
  logs page, planned in #67).

```bash
journalctl -k | grep -E "invoked oom-killer|Out of memory|Memory cgroup out of memory|oom_reaper"
```

Usual causes on a compute: RAM overcommit (`ram_allocation_ratio` above 1.0), not enough `reserved_host_memory_mb`
for the host's own services, or huge pages taken out of the normal memory.

### KVM

From [`arch/x86/kvm/vmx/vmx.c`](https://github.com/torvalds/linux/blob/v7.3/arch/x86/kvm/vmx/vmx.c),
[`svm/svm.c`](https://github.com/torvalds/linux/blob/v7.3/arch/x86/kvm/svm/svm.c) and
[`x86.c`](https://github.com/torvalds/linux/blob/v7.3/arch/x86/kvm/x86.c), each prefixed with the module name
(`kvm_intel:`, `kvm_amd:` or `kvm:`):

| Message (format) | Meaning |
|---|---|
| `kvm_intel: VMX not enabled (by BIOS) in MSR_IA32_FEAT_CTL on CPU <n>` | Virtualisation is disabled in the firmware: KVM cannot load. Nova then cannot start `kvm` instances on this host |
| `kvm_intel: VMX not supported by CPU <n>`, `kvm_amd: SVM not supported by CPU <n>` | The CPU (or the outer hypervisor, for a nested lab) does not expose hardware virtualisation |
| (AMD, SVM disabled in the firmware) | `kvm_amd` in v7.3 has no message of its own for this case, unlike Intel's `VMX not enabled (by BIOS)`; its support checks only log `SVM not supported by CPU <n>` (and `CPU <n> isn't AMD or Hygon`). Older kernels used other wording. Check that `kvm_amd` is loaded and the firmware settings |
| `kvm: SMP vm created on host with unstable TSC; guest TSC will not be reliable` | The host's TSC is not stable; guests may see clock problems |
| `kvm: <instruction> instruction emulated as NOP!` | A guest used an instruction KVM only pretends to run; usually harmless, logged once |

A KVM failure inside a running guest (`KVM internal error. Suberror: <n>`) is printed by QEMU into the instance log,
not by the kernel.

```bash
journalctl -k | grep -E "kvm(_intel|_amd)?:"
lsmod | grep -E "^kvm"            # kvm and kvm_intel or kvm_amd should be loaded
```

### NIC link flaps

Link up and down messages come from the NIC driver, so their wording depends on the driver. Two examples from
Linux v7.3:

| Driver | Messages (format) |
|---|---|
| Intel `ixgbe` ([`ixgbe_main.c`](https://github.com/torvalds/linux/blob/v7.3/drivers/net/ethernet/intel/ixgbe/ixgbe_main.c)) | `NIC Link is Up <speed>, Flow Control: <fc>, EEE: <eee>` and `NIC Link is Down` |
| Mellanox `mlx5` ([`en_main.c`](https://github.com/torvalds/linux/blob/v7.3/drivers/net/ethernet/mellanox/mlx5/core/en_main.c)) | `<interface>: Link up` and `<interface>: Link down` |

The generic IPv6 line `ADDRCONF(NETDEV_CHANGE): <interface>: link becomes ready` is a debug message in v7.3
([`addrconf.c`](https://github.com/torvalds/linux/blob/v7.3/net/ipv6/addrconf.c)): do not rely on it.

```bash
ethtool -i <interface>                                     # which driver, to know which messages to expect
journalctl -k | grep -iE "<interface>.*link (is )?(up|down)"
ip -s link show <interface>                                # carrier changes, errors and drops counters
```

A link that flaps under the tunnel or provider network makes instances lose packets while Neutron and Nova see
nothing.

### Bonding and LACP

This section is about bonds built with the Linux bonding driver. A bond built inside Open vSwitch (a port with
several interfaces in `ovs-vswitchd`) writes nothing to the kernel log: read `ovs-appctl bond/show`,
`ovs-appctl lacp/show` and `ovs-vswitchd.log` instead (see the Open vSwitch logs page, planned in #70).

The bonding driver prefixes per-member messages with `<bond>: (slave <interface>): `
([`include/net/bonding.h`](https://github.com/torvalds/linux/blob/v7.3/include/net/bonding.h)). From
[`bond_main.c`](https://github.com/torvalds/linux/blob/v7.3/drivers/net/bonding/bond_main.c) and
[`bond_3ad.c`](https://github.com/torvalds/linux/blob/v7.3/drivers/net/bonding/bond_3ad.c):

| Message (format) | Meaning |
|---|---|
| `link status down for interface, disabling it in <n> ms` (`for active interface` or `for backup interface` in active-backup mode) | A member lost its link; the bond waits `downdelay` (logged only when `downdelay` is set) |
| `link status up again after <n> ms` | It came back during the delay: a flap |
| `link status definitely down, disabling slave` | The member is out of the bond |
| `link status definitely up, <speed> Mbps <duplex> duplex` | The member is back |
| `making interface the new active one` | Active-backup failover |
| `<bond>: now running without any active interface!` | No member is usable: the host lost that network |
| `<bond>: Warning: No 802.3ad response from the link partner for any adapters in the bond` | 802.3ad (LACP) mode, and the switch does not answer LACP: switch ports not configured as a LAG, or misconfigured |

```bash
journalctl -k | grep -E "bond[0-9]*:|802.3ad"
cat /proc/net/bonding/<bond>        # mode, MII status per member, LACP partner details in 802.3ad mode
```

### MTU and fragmentation hints

The kernel does not log most MTU drops. One exception matters with ML2/OVS and the kernel datapath: a packet larger
than the MTU of the output port, and not GSO, is dropped and counted, with a rate-limited warning
([`net/openvswitch/vport.c`](https://github.com/torvalds/linux/blob/v7.3/net/openvswitch/vport.c)):

```text
<interface>: dropped over-mtu packet: <length> > <mtu>
```

The drop also increments the `tx_errors` counter of that interface. When the logs are silent, use the counters and a
test with the "do not fragment" bit, as in the
[tunnel MTU mismatch runbook](../networking/tunnel-mtu-mismatch-large-packets-dropped.md):

```bash
journalctl -k | grep "dropped over-mtu packet"
ip -s link show <interface>      # TX errors growing on the tunnel or provider interface
```

### Disk and filesystem errors

Block layer errors, from `blk_print_req_error` in
[`block/blk-mq.c`](https://github.com/torvalds/linux/blob/v7.3/block/blk-mq.c), with the error names of
[`blk-core.c`](https://github.com/torvalds/linux/blob/v7.3/block/blk-core.c) (`I/O`, `timeout`, `critical medium`,
`critical target`, `recoverable transport`, …):

```text
<error> error, dev <disk>, sector <n> op 0x<n>:(<READ|WRITE|...>) flags 0x<n> phys_seg <n> prio class <n>
```

Filesystem errors:

| Message (format) | Source | Meaning |
|---|---|---|
| `EXT4-fs error (device <dev>): <function>:<line>: comm <command>: <detail>` | [`fs/ext4/super.c`](https://github.com/torvalds/linux/blob/v7.3/fs/ext4/super.c) | ext4 found an inconsistency |
| `EXT4-fs (<dev>): Remounting filesystem read-only` | same | With `errors=remount-ro`, ext4 stops writing. On the instances directory, every instance that writes to a local disk then fails |
| `<why> (0x<flags>) detected at <function> (<file>:<line>).  Shutting down filesystem.` then `Please unmount the filesystem and rectify the problem(s)` | [`fs/xfs/xfs_fsops.c`](https://github.com/torvalds/linux/blob/v7.3/fs/xfs/xfs_fsops.c) | XFS shut itself down. `<why>` is `Log I/O Error`, `Metadata I/O Error`, `Corruption of in-memory data`, `Corruption of on-disk metadata` or `Block device removal` |

The `<disk>` can be a local disk (the instances directory, the root filesystem) or a remote volume attached through
iSCSI or multipath (`sd*`, `dm-*`). For the iSCSI and multipath side, see the host storage logs page (planned). A full disk is not an
error line in the kernel log; for that case see
[compute disk full, instance paused](../lifecycle/compute-disk-full-instance-paused.md).

```bash
journalctl -k -p err | grep -E "error, dev |EXT4-fs|XFS"
```

### Conntrack table full

Security groups rely on connection tracking (iptables-based firewall drivers with ML2/OVS, and the OVS conntrack
actions of the `openvswitch` firewall driver and of OVN). When the table is full, new connections are dropped,
from [`nf_conntrack_core.c`](https://github.com/torvalds/linux/blob/v7.3/net/netfilter/nf_conntrack_core.c):

```text
nf_conntrack: table full, dropping packet
nf_conntrack: table full in netns <inode>, dropping packet
```

The second form names a network namespace other than the host's. The limit is `net.netfilter.nf_conntrack_max`,
which defaults to `nf_conntrack_buckets`
([`nf_conntrack-sysctl.rst`](https://github.com/torvalds/linux/blob/v7.3/Documentation/networking/nf_conntrack-sysctl.rst)):

```bash
journalctl -k | grep "nf_conntrack: table full"
sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max
```

The symptom is random: established connections keep working, new ones fail on every instance of the host.

### Hung tasks

When a task stays in uninterruptible sleep (D state) longer than `kernel.hung_task_timeout_secs`, the hung task
detector reports it ([`kernel/hung_task.c`](https://github.com/torvalds/linux/blob/v7.3/kernel/hung_task.c),
[sysctl documentation](https://github.com/torvalds/linux/blob/v7.3/Documentation/admin-guide/sysctl/kernel.rst)):

```text
INFO: task <command>:<pid> blocked for more than <n> seconds.
      <tainted flags> <kernel release> <version>
"echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
... stack trace of the task ...
```

Linux v7.3 adds ` in I/O wait` after `blocked` when the task waits on I/O; kernels without that change print the line
without it.
After `kernel.hung_task_warnings` reports (10 by default), the detector logs
`Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings` and stops reporting.

On a compute, a hung `qemu-kvm` or `nova-compute` task almost always waits on storage: an NFS server that stopped
answering, an iSCSI path that is gone without a working multipath, a Ceph cluster that blocks I/O. The stack trace
shows where (`nfs_`, `rpc_`, `iscsi_`, `blk_`, `rbd` frames). Do not follow the hint in the message to disable it.

```bash
journalctl -k | grep -E "blocked (in I/O wait )?for more than|hung task reports are suppressed"
ps -eo pid,stat,comm | awk '$2 ~ /D/'          # tasks in D state now
```

## A short reading order

1. Note the UTC time of the symptom from the OpenStack side (instance stop, port down, volume error).
2. `journalctl -k -o short-iso-precise --since <time - 10 min> --until <time + 5 min>` on the compute.
3. Search the families above in this order: OOM, hung tasks and disk errors (instances stop or freeze), then links,
   bonds, conntrack and MTU (instances lose network).
4. If the host rebooted, use `journalctl -k -b -1`; if it says the boot is not found, the journal is not persistent.

## Validity and limits

- Written from upstream source and documentation only: Linux v7.3, systemd's `journalctl` manual (main branch) and
  util-linux's `dmesg` manual (master). Every message above is a format string from that source, not a captured line.
  Distribution kernels (often older, with backports) can word some messages differently; the stable fragments in the
  `grep` patterns were chosen to match across versions, but this was not checked kernel by kernel.
- Not run on a lab yet. A lab capture is requested of some of these events, injected safely on a lab compute: an OOM kill
  of a test QEMU process, a filled conntrack table, a bond member taken down.
- NIC driver messages differ by driver: only `ixgbe` and `mlx5` are quoted. Check your driver with `ethtool -i`.
- The causes listed for the OOM killer and hung tasks (overcommit, storage) are the usual ones on a compute, given
  as directions to look, not as an exhaustive list.

---

*Provenance: written by agent:ozarland-author, from upstream documentation and source code (Linux kernel, systemd,
util-linux). Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*
