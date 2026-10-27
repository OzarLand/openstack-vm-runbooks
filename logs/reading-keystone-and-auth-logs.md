---
title: Reading the Keystone logs, and the authentication lines every service writes (401, 503, Fernet)
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [keystone, memcached, nova, neutron]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode
  network: any
  storage: any
  os: any
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://opendev.org/openstack/keystone/src/branch/stable/2026.1/keystone/server/flask/application.py
  - https://opendev.org/openstack/keystone/src/branch/stable/2026.1/keystone/token/provider.py
  - https://opendev.org/openstack/keystone/src/branch/stable/2026.1/keystone/token/token_formatters.py
  - https://opendev.org/openstack/keystone/src/branch/stable/2026.1/keystone/common/fernet_utils.py
  - https://opendev.org/openstack/keystonemiddleware/src/branch/stable/2026.1/keystonemiddleware/auth_token/__init__.py
  - https://opendev.org/openstack/keystonemiddleware/src/branch/stable/2026.1/keystonemiddleware/auth_token/_identity.py
  - https://opendev.org/openstack/oslo.log/src/branch/stable/2026.1/oslo_log/_options.py
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Reading the Keystone logs, and the authentication lines every service writes (401, 503, Fernet)

An authentication problem shows in two places: Keystone's own log, and the `keystonemiddleware` lines in the log of
every service that validates tokens (Nova, Neutron, Glance, Cinder, Placement…). The two together tell apart a bad
user token, a service whose own credentials are broken, an unreachable Keystone, and Fernet keys that differ between
Keystone nodes. Messages are quoted from Keystone and `keystonemiddleware` 2026.1.

The fixes for expired certificates and inconsistent Fernet keys are in
[every API call fails at once](../diagnostics/expired-certificates-and-fernet-keys-break-every-api-call.md).

## Where the logs are

**Kolla-Ansible:** Keystone's application log is `/var/log/kolla/keystone/keystone.log` (`keystone.conf.j2`), and in
2026.1 its HTTP requests are in `/var/log/kolla/keystone/keystone-uwsgi.log` (the `service-uwsgi-config` role), on
each controller. The middleware lines of a service are in that service's own log (`nova-api.log`,
`neutron-server.log`, `glance-api.log`, …).

**Any:** `/var/log/keystone/` or the journal, depending on the packaging.

## What hides by default

Every service pins `keystonemiddleware` to `WARN` through `oslo.log`'s default `default_log_levels`
(`keystonemiddleware=WARN`, `keystoneauth=WARN` in `_options.py`). So the middleware's `INFO` lines below
(`Rejecting request`, `Identity server rejected authorization`) do **not** appear unless you lower that level for the
service; its `WARNING`, `ERROR` and `CRITICAL` lines do. `debug = true` alone does not lower a pinned library. Plan
the investigation around the visible lines. If you need the rest, you can set `default_log_levels` for one service.

> ⚠️ **State-changing.** Changing `default_log_levels` means editing the service's configuration and restarting it
> on that host, with the same impact as the debug procedure in
> [turning debug on for one service](where-openstack-logs-are-and-how-to-read-a-line.md#turning-debug-on-for-one-service-then-off).
> Undo: remove the `default_log_levels` line and restart the service again.

Setting it **replaces the whole default list**. Copy the defaults from `oslo_log/_options.py`, **keep the entries the
service adds to them** (Nova, for example, appends `glanceclient=WARN` and `oslo.privsep.daemon=INFO` in
`nova/config.py`), and change only `keystonemiddleware=WARN` to `keystonemiddleware=INFO`.

## In Keystone's log

| Line (message text) | Level | What it means |
|---|---|---|
| `Authorization failed. <message> from <client address>` | WARNING | Keystone answered 401 to a request: a wrong password, an unknown user, a token Keystone refuses for authentication. `<message>` is the exception text, often `The request you have made requires authentication.` The client address names the caller (behind HAProxy, the address HAProxy forwards). |
| `Unable to validate token: <error>` | DEBUG | Validation of a token failed inside Keystone; the caller gets 404 (`TokenNotFound`). Visible with debug on. |
| `Unexpected error or malformed token determining token expiry: <token>` with a traceback | ERROR | A token Keystone decrypted but could not read. |
| `Either [<group>] key_repository does not exist or Keystone does not have sufficient permission to access it: <path>` | ERROR | Keystone cannot read its Fernet (or credential) key repository on this node: it cannot issue or validate tokens here. |
| `key_repository is world readable: <path>` | WARNING | Permissions too open on the key directory. |
| `Starting key rotation with <n> key files: [...]`, `Current primary key is: <n>`, `Next primary key will be: <n>` | INFO | A Fernet key rotation ran on this node. Compare the times and key numbers across Keystone nodes: they must end with the same keys. |

Sources: `keystone/server/flask/application.py`, `keystone/token/provider.py`, `keystone/common/fernet_utils.py`.

When Keystone cannot decrypt a token with any of its keys, the Fernet formatter raises `Could not recognize Fernet
token` (`token_formatters.py`); the Fernet provider turns it into `TokenNotFound` (`token/providers/fernet/core.py`),
and Keystone logs `NotFound` errors at DEBUG (`application.py`). So, **with debug on**, a node with different keys
logs `Could not recognize Fernet token` for tokens issued by another node; at the default level there is no
dedicated line, only the 404 to the caller. That string is the one to look for in a key-mismatch incident.

```bash
# Kolla-Ansible, on each controller
grep -c "Authorization failed" /var/log/kolla/keystone/keystone.log
grep "Authorization failed" /var/log/kolla/keystone/keystone.log | awk '{print $NF}' | sort | uniq -c | sort -rn | head
#   who gets refused most (last field: client address)
```

## In the other services: keystonemiddleware

Every API service validates the token of each request with `keystonemiddleware`, which asks Keystone (and caches the
answer in memcached).

| Line (message text) | Level | Visible by default | What it means | Client gets |
|---|---|---|---|---|
| `Authorization failed for token` | WARNING | yes | Keystone said the **user's** token is not valid (expired, revoked, or unknown to this Keystone node). | 401 |
| `Identity response: <body>` | WARNING (with the 401 path and the `Bad response code` path) / INFO (with the 404 path) | partly | Keystone's answer body, logged alongside the lines above and below. | — |
| `Unable to validate token: <error>` | CRITICAL | yes | The middleware could not get an answer it trusts: Keystone unreachable or timing out (`ConnectFailure`, `RequestTimeout`, `DiscoveryFailure`), or the **service's own** credentials refused (`Identity server rejected authorization necessary to fetch token data`), or an HTTP error from Keystone (`Failed to fetch token data from identity server`, after the `Bad response code` line). | 503, `The Keystone service is temporarily unavailable.` (with `delay_auth_decision = true` the middleware does not answer itself: it marks the token invalid and the service decides) |
| `Bad response code while validating token: <status> <message>` | ERROR | yes | Keystone answered with an unexpected HTTP error (for example 500 or 503). | 503 |
| `Identity server rejected authorization` | INFO | no | Keystone refused the service's own token (`[keystone_authtoken]` user, password or project wrong). The middleware retries once, then logs the CRITICAL line above. | — |
| `Rejecting request` | INFO | no | The request is refused because the token is missing or invalid. | 401 |

Sources: `keystonemiddleware/auth_token/__init__.py`, `keystonemiddleware/auth_token/_identity.py`.

The useful split:

- **401 for users, WARNING `Authorization failed for token` in the services:** the tokens are the problem. One user:
  their token or session. Every user, right after a key rotation or on some Keystone nodes only: Fernet keys differ
  between Keystone nodes.
- **503 for users, CRITICAL `Unable to validate token` in the services:** the services cannot use Keystone. Read the
  error in the line: a connection error points at Keystone, HAProxy or the network; `Identity server rejected
  authorization necessary to fetch token data` points at that service's `[keystone_authtoken]` credentials (a changed
  password not deployed everywhere).

```bash
# Kolla-Ansible, on a controller: which services complain, and how
grep -l "Unable to validate token" /var/log/kolla/*/*.log
grep -h "Unable to validate token" /var/log/kolla/*/*.log | sed 's/.*Unable to validate token: //' | sort | uniq -c
grep -c "Authorization failed for token" /var/log/kolla/nova/nova-api.log
```

## Clock and certificates

Token expiry is checked against Keystone's clock; a Keystone node whose clock is wrong rejects tokens as expired or
accepts expired ones ([clock skew](../diagnostics/compute-clock-skew.md) covers the compute case; a controller drift is
more serious). An expired TLS certificate shows as connection errors (`ConnectFailure`, SSL errors) in the
`Unable to validate token` line rather than as a Keystone message: see
[every API call fails at once](../diagnostics/expired-certificates-and-fernet-keys-break-every-api-call.md).

## Reading an authentication incident in order

1. **What does the client get?** 401 or 503: they are different problems.
2. **Which services log it, and since when?** `Authorization failed for token` (WARNING) or `Unable to validate token`
   (CRITICAL), first occurrence time.
3. **Keystone's side at the same time:** `Authorization failed. … from <address>` for refused logins, key repository
   errors, a rotation.
4. **Across Keystone nodes:** the same key files on every node, the same clock.
5. **Below:** HAProxy's `keystone` backend and memcached (the HAProxy, keepalived and memcached page of this folder,
   once published).

## Validity and limits

- Every message is quoted from Keystone, `keystonemiddleware` and `oslo.log` 2026.1. **None was captured on a lab.**
  A lab request asks for real samples.
- Which middleware line appears for a given failure follows from the source paths read here; the client status codes
  (401, 503) are the ones the middleware raises in those paths.
- Federation, application credentials, OAuth2 and the `external_oauth2_token` middleware are not covered.
- The behaviour of a node with mismatched Fernet keys is described from the code path, not observed.

## Related

- [Every API call fails at once: expired certificates or Fernet keys](../diagnostics/expired-certificates-and-fernet-keys-break-every-api-call.md)
- [Where the OpenStack logs are, and how to read one log line](where-openstack-logs-are-and-how-to-read-a-line.md)
- [Logs index](README.md)

---

*Provenance: written by agent:ozarland-author, from Keystone, keystonemiddleware and oslo.log 2026.1 source code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=logs%2Freading-keystone-and-auth-logs.md&title=Feedback%3A+logs%2Freading-keystone-and-auth-logs.md):
your release, your deployment method, and what happened.
