---
title: Every API call fails at once. Expired TLS certificates or inconsistent Keystone Fernet keys
kind: diagnostic
status: draft
openstack_releases: [unrecorded]
components: [keystone, haproxy, rabbitmq, mariadb, memcached]
deployment_methods: [any]
environment:
  topology: any
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/keystone/latest/admin/fernet-token-faq.html
  - https://docs.openstack.org/kolla-ansible/latest/admin/tls.html
last_verified: 2026-10-18
---

# Every API call fails at once. Expired TLS certificates or inconsistent Keystone Fernet keys

Two components of the control plane age silently until they fail completely, and when they fail, nothing works:
no VM creation, no listing, no console, often no Horizon. Running instances are unaffected. Both failures are dated
in advance and entirely preventable, which is why they are embarrassing and why they belong on the first screen of
any monitoring.

## Symptom

- Certificates: every client gets a TLS error (`certificate verify failed`, `certificate has expired`) or the
  load balancer marks backends down because its health checks fail TLS. Services that talk to each other over TLS
  (Nova to Neutron, Keystone to the database or RabbitMQ when TLS is enabled there) log SSL errors. The failure
  starts at an exact timestamp.
- Fernet keys: `openstack token issue` works on some Keystone nodes and not others, or tokens just issued are
  rejected (`401 Unauthorized`, `The request you have made requires authentication`) by the next service in the
  chain. Keystone logs `Could not recognize Fernet token` or key-loading errors. It starts right after a key
  rotation or a Keystone node rebuild.

## Impact

Complete loss of the control plane for users and automation. Running instances keep running. Monitoring based on
the APIs goes blind at the same moment, so the alert often comes from users.

## Quick checks

```bash
# Expiry of the certificate behind each endpoint (repeat for internal and public VIPs, ports 5000, 8774, 9696, 9292, 8776, 8780...)
openssl s_client -connect <internal-vip>:5000 </dev/null 2>/dev/null | openssl x509 -noout -enddate -subject -issuer
# RabbitMQ and MariaDB if TLS is enabled there
openssl s_client -connect <controller>:5671 </dev/null 2>/dev/null | openssl x509 -noout -enddate
openssl s_client -connect <controller>:3306 -starttls mysql </dev/null 2>/dev/null | openssl x509 -noout -enddate

# Fernet keys identical on every Keystone node (same files, same checksums)
# Kolla-Ansible: docker exec keystone ls -l /etc/keystone/fernet-keys ; md5sum of each key
# Any: ls -l /etc/keystone/fernet-keys && md5sum /etc/keystone/fernet-keys/*
# Compare across nodes. Key "0" is the staged key, the highest number is the primary.

# Does a token issued on one node validate on another?
openstack token issue -f value -c id
```

## Diagnosis

**Certificates.** The chain of trust has several links, each with its own expiry: the endpoint certificates on
the load balancer, the backend certificates if TLS terminates on the services, the CA certificate that every
service and every agent trusts (`[DEFAULT] cafile`, system CA bundle), and the client certificates used for
RabbitMQ or MariaDB when mutual TLS is on. Check each; an expired CA breaks everything at once, an expired backend
certificate breaks one service. On Kolla-Ansible, `kolla-ansible certificates` generates and `kolla-ansible deploy`
or `reconfigure` distributes them; look at the paths in `/etc/kolla/certificates/`.

**Fernet keys.** Keys do not expire, but rotation must be done correctly and the result must be identical on every
Keystone node. A rotation that removes a key still needed to validate tokens in flight, or that leaves nodes with
different key sets, breaks authentication for the whole platform at once. `keystone-manage fernet_rotate` must run
on one node and the directory must then be distributed to the others (Kolla-Ansible does this with its
`fernet_token_expiry` and rotation schedule inside the `keystone_fernet` container). A rebuilt node that generated
its own keys with `fernet_setup` instead of receiving the shared set is the other classic case.

## Possible causes

| Cause | How to tell |
|---|---|
| Endpoint or CA certificate expired | `openssl x509 -enddate` in the past; failure started at that instant |
| Certificate renewed but not distributed to agents or backends | Load balancer fine, services log SSL errors to each other; one node or service differs |
| Fernet keys differ between Keystone nodes | Checksums differ; tokens validate only on some nodes |
| Rotation removed a key still in use | Fails right after rotation; recovers for new tokens, old ones rejected |

## What to do next

This page is read-only. Fixing is a deployment-tool procedure: regenerate and distribute certificates, or
distribute one consistent Fernet key set to every Keystone node and restart Keystone, with the tool you deployed
with (Kolla-Ansible: `kolla-ansible certificates` then `reconfigure`; Fernet keys live in the `keystone_fernet_tokens`
volume on each controller, and `kolla-ansible reconfigure -t keystone` re-runs the Keystone deployment that
distributes them). After the fix, every service and agent that cached the old certificate may need
a restart.

## Prevention, which is the actual runbook

- Put the expiry date of every certificate on the control plane on the first monitoring screen, with an alert at
  30 days. Include RabbitMQ, MariaDB, Ceph dashboards and anything else behind TLS, not only the OpenStack VIPs.
- Put certificate rotation in the maintenance calendar, and rehearse it on a lab: a rotation that has never been
  rehearsed is a change with an unknown blast radius.
- Monitor the age and consistency of the Fernet keys across Keystone nodes. A platform that has never rotated its
  keys has never tested that it can.
- Include both in recovery exercises.

## Validity and limits

Derived from an operator's account of long-lived platforms where certificate expiry was one of the most common
causes of a complete control-plane outage; release not recorded. Paths and commands are given for Kolla-Ansible;
other deployment tools manage certificates and Fernet keys their own way.

## Related

- [VM creation fails platform-wide: check the shared dependencies first](vm-creation-fails-platform-wide-check-shared-dependencies.md).
- Source: *OpenStack, the Day After Tomorrow* (S. Zaouam), chapters 8 and 17.
