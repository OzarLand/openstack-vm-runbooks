[← openstack-vm-runbooks](../README.md)

# Logs: where they are and how to read them

How to find, read and interpret the logs of OpenStack services and of the system around them (infrastructure
services, hypervisor, kernel), so that an operator can go from a symptom to the line that explains it. Each page
cites the upstream documentation or source code it comes from; real log lines come from lab runs and are redacted.

These pages explain how to read; the fixes are in the runbooks of the other folders, which the pages link to.

## Pages

| Page | Covers | Releases | Status |
|---|---|---|---|
| [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md) | Kolla-Ansible and package installs, the oslo.log line format, request ids, levels, debug on and off | 2026.1 | draft |
| [Reading the Heat logs](reading-heat-logs.md) | Stack and resource failure lines, the ResourceFailure status reason, following the physical id into the service that failed | 2026.1 | draft |

## Planned

1. Tracing one request across services: request ids, global request ids, instance, port and volume ids; grep and
   journalctl recipes; time correlation and clock skew.
2. Per service, what normal looks like and what the common error lines mean: Nova, Placement, Neutron and its
   agents, OVN, Keystone, Glance, Cinder, Heat, Octavia.
3. Infrastructure logs: RabbitMQ, MariaDB/Galera, HAProxy, keepalived, memcached.
4. Hypervisor and system logs: libvirt and QEMU, Open vSwitch, the kernel, systemd and the journal, the container
   engine, chrony, iSCSI and multipath, Ceph clients.
5. A dictionary of frequent error messages: what each means, its usual causes, and the next log to read.
