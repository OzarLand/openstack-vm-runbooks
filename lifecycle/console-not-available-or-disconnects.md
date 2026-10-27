---
title: Instance console (noVNC or SPICE) does not open, fails at once, or shows "invalid or has expired"
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [nova, libvirt, haproxy, horizon]
deployment_methods: [kolla-ansible, any]
environment:
  topology: any multinode
  network: any
  storage: any
  os: any
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/admin/remote-console-access.html
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#vnc
  - https://docs.openstack.org/nova/2026.1/configuration/config.html#consoleauth.token_ttl
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/compute/nova-guide.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Instance console (noVNC or SPICE) does not open, fails at once, or shows "invalid or has expired"

## Symptom

One of these, from the dashboard's console tab or from the URL of `openstack console url show <server-id>`:

- the API refuses to give a URL: `Unavailable console type novnc.`, `Instance <id> is not ready`, or an error about
  the compute service;
- the URL opens a blank page, or the browser cannot reach it;
- the noVNC page opens, then reports at once that it could not connect or was disconnected, or the session drops
  after a while;
- an old URL, or a URL opened a while after it was created, fails: the proxy logs
  `The token '***' is invalid or has expired`.

## Impact

One instance, or every console of a compute or of the platform, depending on the cause. The instance itself is not
affected: only its console. Users who rely on the console (no SSH key, broken network inside the guest) cannot reach
their instance.

## Quick checks

Read-only.

```bash
# 1. Can the API build a console URL? (as the instance's owner or an admin)
openstack console url show <server-id>                 # noVNC by default; --spice for SPICE
# 2. The instance and its compute
openstack server show <server-id> -c status -c OS-EXT-SRV-ATTR:host -c OS-EXT-SRV-ATTR:instance_name
openstack compute service list --service nova-compute --host <compute>
```

The console proxy's log is where a failing connection says why:

**Kolla-Ansible:** on the controllers, `sudo tail -n 50 /var/log/kolla/nova/nova-novncproxy.log` (SPICE:
`nova-spicehtml5proxy.log`). The `nova_novncproxy` containers must be running and healthy.

**Any:** the log of `nova-novncproxy` (or `nova-spicehtml5proxy`).

On the compute, the guest's console server, from its libvirt definition:

**Kolla-Ansible:** `sudo docker exec nova_libvirt virsh vncdisplay <instance-name>` (SPICE:
`virsh domdisplay <instance-name>`)

**Any:** `sudo virsh vncdisplay <instance-name>` (SPICE: `sudo virsh domdisplay <instance-name>`)

This page follows noVNC, Kolla-Ansible's default. With `nova_console: spice`, read `nova_spicehtml5proxy` (port 6082),
`[spice] html5proxy_base_url` and `openstack console url show --spice <server-id>` wherever the page says
`nova_novncproxy`, port 6080, `[vnc] novncproxy_base_url` and `--novnc`.

## Diagnosis

### How a console connection works (Nova 2026.1)

1. `openstack console url show` (or the dashboard) asks the API, which asks the instance's compute. The compute reads
   the guest's VNC port from its libvirt definition, takes its own `[vnc] server_proxyclient_address` and that port,
   and creates the token itself: it stores it in the cell database with an expiry of `[consoleauth] token_ttl`
   (`get_vnc_console` in `nova/compute/manager.py`). The API returns `novncproxy_base_url` with the token.
2. The browser opens that URL. The proxy (`nova-novncproxy`) checks the token, asks the compute that the port is still
   the instance's, checks the browser's `Origin` header, then connects to `server_proxyclient_address:<port>`.
3. The token is valid `[consoleauth] token_ttl` seconds (600 by default, read on the **compute** when the token is
   created): a URL must be opened within 10 minutes. The proxy only checks the stored expiry.
4. An open session stays up until the browser closes it, unless `[consoleauth] enforce_session_timeout` is true (the
   proxy then closes it when the token expires, and logs `<n> seconds to terminate connection.`), or a load balancer
   cuts it (Kolla-Ansible's HAProxy: `timeout tunnel 1h`).

### What Kolla-Ansible 2026.1 configures

- `nova_console: novnc` by default (`spice` or `none` otherwise).
- On every compute, `server_listen` and `server_proxyclient_address` are the host's `api_interface_address`; the
  proxies listen on port 6080 (`nova_novncproxy_port`) of each controller.
- `novncproxy_base_url` is built from `nova_novncproxy_fqdn` (default: `kolla_external_fqdn`), the public protocol and
  `nova_novncproxy_public_port`, path `/vnc_lite.html`. With `nova_console: spice`, Kolla-Ansible sets
  `[vnc] enabled = false` and `[spice] enabled = true`.
- HAProxy publishes the proxy on the internal and external VIPs, with `timeout tunnel 1h`.

### Messages and where they come from

| Message | Where | Meaning |
|---|---|---|
| `Unavailable console type novnc.` (or `vnc`) | API answer | VNC is disabled on that compute (`[vnc] enabled = false`, as on a SPICE platform), or the guest's libvirt definition has no VNC device |
| `Invalid console type <type>` | API answer | The type requested is not one the compute's console handler knows |
| `Instance <id> is not ready` | API answer | The instance has no host yet, or its guest is missing while it is still `BUILDING` |
| HTTP 404, instance not found | API answer | The instance is `ACTIVE` but its compute does not find the guest in libvirt |
| `The token '***' is invalid or has expired` | proxy log | Token older than `token_ttl`, or the console port changed (the instance moved, rebooted onto another port) |
| `Origin header does not match this host.` | proxy log | The browser's address is not the proxy's host name, nor in `[console] allowed_origins` |
| `Origin header protocol does not match this host.` | proxy log | `https` page, `http` URL in `novncproxy_base_url`, or the reverse (upstream FAQ) |
| `connecting to: <address>:<port>`, then a connection error | proxy log | The proxy cannot reach the compute's VNC address and port |

## Possible causes

### Cause 1: the token expired or no longer matches

The URL was opened more than `token_ttl` seconds after it was created, reused later, or the instance moved since.

### Cause 2: the URL is wrong for the browser

`novncproxy_base_url` names an address or protocol the user's browser cannot use: an internal address, `http` behind
an `https` front end, a host name that does not resolve outside. With Kolla-Ansible, a wrong `kolla_external_fqdn` or
TLS setting gives this. The page does not load, or fails on the Origin checks.

### Cause 3: the proxy cannot reach the compute

The controllers cannot reach the compute's `server_proxyclient_address` on the VNC ports (5900 and above, one per
guest): a firewall, a routing change, or a wrong address after the compute's interface changed.

### Cause 4: the compute or the guest

`nova-compute` is down (no URL can be built), VNC is disabled on that compute, or the guest's libvirt definition has
no VNC device (for example a guest defined while VNC was disabled on that compute).

### Cause 5: the proxy service is down

The `nova_novncproxy` containers stopped or are unhealthy, or HAProxy has no healthy backend for them: every console
fails.

### Cause 6: the session is cut after a while

The console works, then drops: after 1 hour through Kolla-Ansible's HAProxy (`timeout tunnel 1h`), or at the token's
expiry when `[consoleauth] enforce_session_timeout` is true (proxy log `<n> seconds to terminate connection.`).

## Resolution

### For cause 1: get a new URL

Run `openstack console url show <server-id>` again (or reopen the console tab), and open it at once. If users often
need longer, raise `[consoleauth] token_ttl` on the **computes**, which create the tokens.

**Kolla-Ansible:** in `/etc/kolla/config/nova/nova-compute.conf`, `[consoleauth] token_ttl = <seconds>`, then
`kolla-ansible reconfigure -i <inventory> --tags nova --limit <computes>`. The `nova` tag also runs the `nova-cell`
play on conductor and proxy hosts: `--limit` must name only compute hosts.

> ⚠️ **State-changing.** Restarts `nova_compute` on the computes; running instances are not affected. A longer
> lifetime also means a leaked console URL stays usable longer. Undo: remove the override and reconfigure again.

**Any:** set it in `nova.conf` on the computes and restart `nova-compute`.

### For cause 2: fix the public URL

**Kolla-Ansible:** for the console alone, set `nova_novncproxy_fqdn` (it defaults to `kolla_external_fqdn`) and
`nova_novncproxy_public_port` in `globals.yml`, then:

> ⚠️ **State-changing.** `reconfigure` rewrites `nova.conf`, including `novncproxy_base_url` on the computes, and
> restarts the Nova containers whose configuration changed; running instances are not affected. Undo: revert
> `globals.yml` and reconfigure again.

```bash
kolla-ansible reconfigure -i <inventory> --tags nova
```

If the platform-wide settings themselves are wrong (`kolla_external_fqdn`, external TLS, certificates), every public
endpoint is affected, not only the console: that is outside this runbook, and must not be applied with `--tags nova`
only, which would leave the other services and the Keystone catalog inconsistent.

**Any:** set `[vnc] novncproxy_base_url` on the computes to the address and protocol users reach, and, if the proxy
is published under another name, add it to `[console] allowed_origins` where the proxy runs.

### For cause 3: open the path from the proxies to the compute

From a controller, check that the compute's VNC port answers (`virsh vncdisplay` gives the display number; the port is
5900 plus that number), then open that range between the controllers and the compute's API network.

### For cause 4: fix the compute or the guest

Start `nova-compute` if it is down. If VNC is disabled on that compute (`[vnc] enabled = false`) while the platform
uses noVNC, enable it in that compute's configuration:

> ⚠️ **State-changing.** Changes the compute's `nova.conf` and restarts `nova-compute`; running instances are not
> affected, and keep the definition they have until they are rebuilt (below). Undo: revert the change and restart
> again.

**Kolla-Ansible:** remove the override that disables it (`[vnc] enabled = false`), wherever it is:
`/etc/kolla/config/global.conf`, `/etc/kolla/config/nova.conf`, `/etc/kolla/config/nova/nova-compute.conf`, or the
per-host `/etc/kolla/config/nova/<inventory-hostname>/nova.conf` and
`/etc/kolla/config/nova/<inventory-hostname>/nova-compute.conf`; then
`kolla-ansible reconfigure -i <inventory> --tags nova --limit <compute>`.

**Any:** set `[vnc] enabled = true` in that compute's `nova.conf` and restart `nova-compute`.

If the guest has no VNC device, its definition will only change when Nova rebuilds it: a hard reboot regenerates the
domain from the flavor and image.

> ⚠️ **State-changing.** A hard reboot stops the guest abruptly and starts it again; unsaved data in the guest is lost.
> Agree it with the owner. Undo: none needed.

```bash
openstack server reboot --hard <server-id>
```

### For cause 5: restart the proxies

> ⚠️ **State-changing.** Restarts the console proxy; open consoles are disconnected. Undo: none needed.

**Kolla-Ansible:** `sudo docker restart nova_novncproxy` on each controller.

**Any:** restart `nova-novncproxy`.

### For cause 6: decide how long sessions may last

The 1-hour cut comes from the load balancer; with Kolla-Ansible it is in the `nova-cell` load balancer settings
(`timeout tunnel 1h`), so users reconnect with a new URL. Keep `enforce_session_timeout` if sessions must end with
their token; otherwise, a session ends when the user closes it.

## Verification

```bash
openstack console url show <server-id>
```

Open the URL within the token's lifetime: the guest's screen appears, and the proxy log shows `connect info:` and
`connecting to:` lines for it without an error.

## Validity and limits

- Written from Nova 2026.1 (remote console guide and its FAQ, `nova/console/websocketproxy.py`,
  `nova/compute/manager.py` `get_vnc_console`, `nova/virt/libvirt/driver.py` `get_vnc_console`, `nova/exception.py`)
  and Kolla-Ansible 2026.1 (`nova-cell` `nova.conf.j2`, `group_vars/all/nova.yml`, the `nova-cell` load balancer
  tasks). Not reproduced on a lab.
- SPICE is covered only through the equivalences named under *Quick checks*.
- Not covered: VNC TLS between proxy and compute (`vnc.auth_schemes`), serial consoles, RDP and MKS, and Horizon's
  own configuration.

## Related

- [nova-compute is down on a compute, but its instances are still running](../recovery/nova-compute-down-instances-still-running.md)
- [New instance has an address, but no SSH key](../networking/new-instance-boots-without-metadata.md): often the
  reason users need the console.
- Nova 2026.1: [remote console access](https://docs.openstack.org/nova/2026.1/admin/remote-console-access.html)

---

*Provenance: written by agent:ozarland-author, from the Nova and Kolla-Ansible 2026.1 documentation and source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=lifecycle/console-not-available-or-disconnects.md&title=Feedback%3A+lifecycle/console-not-available-or-disconnects.md):
your release, your deployment method, and what happened.
