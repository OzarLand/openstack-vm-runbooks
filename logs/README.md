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
| [Following one request, instance, port or volume through the logs of every service and host](tracing-one-request-across-services-and-hosts.md) | Which id links which hop, grep and journalctl recipes across hosts, the port-id chain of network-vif-plugged, time zones and clock offset | 2026.1 | draft |
| [Reading the Nova logs. What normal looks like, and what the common lines mean](reading-nova-logs.md) | nova-api, nova-scheduler, nova-conductor, nova-compute: start-up, periodic lines, common error lines and what to read next | 2026.1 | draft |
| [Reading the RabbitMQ logs, and the oslo.messaging lines that go with them](reading-rabbitmq-logs.md) | Broker log format, Khepri or Mnesia, nodes down and partitions, memory and disk alarms, flow control, connection churn and heartbeats, client reconnect lines | 2026.1 | draft |
| [Reading the Neutron logs with ML2/OVS](reading-neutron-logs-ml2-ovs.md) | neutron-server, RPC server and periodic workers, OVS, L3, DHCP and metadata agents: heartbeats, port binding, the vif-plugged notification, common lines | 2026.1 | draft |

## Planned

1. Per service, what normal looks like and what the common error lines mean: Nova, Placement, Neutron and its
   agents, OVN, Keystone, Glance, Cinder, Heat, Octavia.
2. Infrastructure logs: RabbitMQ, MariaDB/Galera, HAProxy, keepalived, memcached.
3. Hypervisor and system logs: libvirt and QEMU, Open vSwitch, the kernel, systemd and the journal, the container
   engine, chrony, iSCSI and multipath, Ceph clients.
4. A dictionary of frequent error messages: what each means, its usual causes, and the next log to read.
