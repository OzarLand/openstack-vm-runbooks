---
title: "Error messages: compute and libvirt. Instance failed to spawn, Connection to libvirt lost"
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [nova, libvirt, qemu]
deployment_methods: [any]
environment:
  topology: any
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by: []
upstream_refs:
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/compute/manager.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/virt/libvirt/host.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/virt/libvirt/driver.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/exception.py
  - https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/src/qemu/qemu_monitor.c
  - https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/src/qemu/qemu_process.c
  - https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/src/cpu/cpu_x86.c
  - https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/src/rpc/virnetsocket.c
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Error messages: compute and libvirt. Instance failed to spawn, Connection to libvirt lost

This page belongs to the dictionary of frequent OpenStack error messages. For each message: where it appears, what it
means, the usual causes, the next log to read, and the runbook that fixes it. The messages are the format strings of
Nova `stable/2026.1` and libvirt v12.8.0, not captured lines; `<…>` marks a value.

The Nova side of these lines is also described on [reading the Nova logs](reading-nova-logs.md); the libvirt and QEMU
side on [reading the libvirt and QEMU logs](libvirt-and-qemu-logs.md).

## `Instance failed to spawn`

- **Where:** ERROR with a traceback in `nova-compute.log`, prefixed by the instance uuid (`LOG.exception` in the build
  resources of [`compute/manager.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/compute/manager.py)).
  It is usually followed by `Failed to build and run instance` (ERROR, with a traceback) and, if the failure can be
  retried elsewhere, `Build of instance <uuid> was re-scheduled: <reason>`; otherwise `Build of instance <uuid>
  aborted: <reason>`.
- **Means:** something failed while preparing the build or while the virt driver created the guest. The message
  itself says nothing about the cause: **the last line of the traceback does**.
- **Common last lines, and where they lead:**

  | Traceback ends with (format) | From | Read next |
  |---|---|---|
  | `libvirt.libvirtError: … QEMU unexpectedly closed the monitor (vm='<name>') …` | libvirt `qemu_monitor.c`: QEMU exited during start-up | The instance log `/var/log/libvirt/qemu/<instance-name>.log` on that compute: QEMU's own error just before (a missing device, a file it cannot open, memory) |
  | `libvirt.libvirtError: … process exited while connecting to monitor …` | libvirt `qemu_process.c`: same, earlier in start-up | Same instance log |
  | `libvirt.libvirtError: … guest CPU doesn't match specification: missing features: <features>` | libvirt `cpu_x86.c` | The CPU model Nova asks for (`[libvirt] cpu_mode`, `cpu_models`) against this host's CPU |
  | `VirtualInterfaceCreateException: Virtual Interface creation failed` | Nova, after the vif plugging timeout | The networking error messages page (planned), and [waiting for vif_plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md) |
  | `PortBindingFailed: Binding failed for port …` | Nova | [Binding failed for port](../networking/port-binding-failed.md) |
  | An image or disk error (`ImageNotFound`, a download error, `No space left on device`) | Nova, Glance, the host | The Glance logs; free space in the instances directory |

  When the build is retried on other computes and fails everywhere, the instance ends with `Exceeded maximum number of
  retries` (see the scheduling error messages page, in review in #123).
- **Usual causes:** the guest cannot start on this host (CPU model, missing device or driver, memory), the network or
  the volume attach failed during the build, the image could not be fetched or written.
- **Fix:** depends on the last line; [Exceeded maximum number of retries](../scheduling/build-fails-exceeded-maximum-number-of-retries.md)
  lists the compute-side errors and where each leads.

## `Connection to libvirt lost: <reason>`

- **Where:** built in [`virt/libvirt/host.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/virt/libvirt/host.py)
  when libvirt closes nova-compute's connection, and logged by the driver as
  `Connection event '0' reason 'Connection to libvirt lost: <reason>'` (INFO, `_handle_conn_event` in
  [`virt/libvirt/driver.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/virt/libvirt/driver.py)). At
  start, the same path logs `Connection event '0' reason 'Failed to connect to libvirt: <error>'`. When the
  connection comes back: `Connection event '1' …`.
- **Means:** nova-compute lost its connection to the libvirt daemon. The driver then **disables the compute service
  automatically**, with a disabled reason prefixed `AUTO: ` (`DISABLE_PREFIX`), so the scheduler stops placing
  instances there; it re-enables the service when the connection returns, but only if the reason still starts with
  `AUTO: ` (an operator's manual disable is kept). Running instances are not affected by the lost connection itself.
- **Related:** calls made while the connection is down fail with `HypervisorUnavailable`, whose message is `Connection to
  the hypervisor is broken on host` ([`exception.py`](https://github.com/openstack/nova/blob/stable/2026.1/nova/exception.py)).
  On the libvirt client side, a broken connection reads `End of file while reading data: …` or `Cannot recv data: …`,
  and a daemon that is not listening `Failed to connect socket to '<path>': …`
  ([`rpc/virnetsocket.c`](https://gitlab.com/libvirt/libvirt/-/blob/v12.8.0/src/rpc/virnetsocket.c)).
- **Read next:** is the libvirt daemon running (`nova_libvirt` on Kolla-Ansible, `virtqemud` or `libvirtd`
  otherwise), and the daemon's own log at that time (restart, crash); `openstack compute service list --long` for the
  `AUTO: ` reason.
- **Usual causes:** the libvirt daemon or its container was restarted, upgraded or crashed; the host is short of
  memory and the daemon was killed.
- **Fix:** [Restarting nova_libvirt does not stop instances](../lifecycle/restarting-nova-libvirt-does-not-stop-instances.md)
  for a planned restart; [compute service disabled or forced down](../scheduling/compute-service-disabled-or-forced-down.md)
  if the service stays disabled; [OOM killer](../lifecycle/instance-shut-off-by-host-oom-killer.md) if the host ran out
  of memory.

## A short reading order

1. `Instance failed to spawn`: read the **last line** of the traceback, then the instance log of QEMU on the same
   compute at the same time.
2. `Connection to libvirt lost`: the libvirt daemon's state and log, then the compute service's disabled reason.

## Validity and limits

- Written from Nova `stable/2026.1` and libvirt v12.8.0 source only. Every message is a format string from that source,
  not a captured line. The libvirt error text inside a Python `libvirtError` traceback is usually prefixed by a libvirt
  error class (for example `internal error:`); the exact prefix was not checked.
- Not run on a lab yet. Captures of a spawn failure caused by a CPU model mismatch, and of a `nova_libvirt` restart with
  the `Connection event` lines, are requested.

---

*Provenance: written by agent:ozarland-author, from upstream source code (Nova, libvirt). Reviewed in its pull request
(see the page history). Checked by a human operator: not yet.*
