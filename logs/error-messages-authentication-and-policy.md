---
title: "Error messages: authentication and policy. 401 requires authentication, 403 policy, service credentials, Keystone unreachable"
kind: diagnostic
status: draft
openstack_releases: ["2026.1"]
components: [keystone, nova, neutron, cinder, glance]
deployment_methods: [any]
environment:
  topology: any
  network: any
  storage: any
  os: any
risk: read-only
confirmed_by: []
upstream_refs:
  - https://github.com/openstack/keystone/blob/stable/2026.1/keystone/exception.py
  - https://github.com/openstack/keystonemiddleware/blob/stable/2026.1/keystonemiddleware/auth_token/__init__.py
  - https://github.com/openstack/keystoneauth/blob/stable/2026.1/keystoneauth1/exceptions/http.py
  - https://github.com/openstack/keystoneauth/blob/stable/2026.1/keystoneauth1/exceptions/catalog.py
  - https://github.com/openstack/keystoneauth/blob/stable/2026.1/keystoneauth1/session.py
  - https://github.com/openstack/keystoneauth/blob/stable/2026.1/keystoneauth1/identity/generic/base.py
  - https://github.com/openstack/oslo.policy/blob/stable/2026.1/oslo_policy/policy.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/exception.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/network/neutron.py
  - https://github.com/openstack/nova/blob/stable/2026.1/nova/conf/service_token.py
  - https://github.com/openstack/neutron/blob/stable/2026.1/neutron/pecan_wsgi/hooks/policy_enforcement.py
  - https://github.com/openstack/neutron/blob/stable/2026.1/neutron/notifiers/nova.py
  - https://github.com/openstack/cinder/blob/stable/2026.1/cinder/exception.py
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Error messages: authentication and policy. 401 requires authentication, 403 policy, service credentials, Keystone unreachable

This page belongs to the dictionary of frequent OpenStack error messages. For each message: where it appears, what it
means, the usual causes, the next log to read, and the runbook that fixes it. The messages are the format strings of
the `stable/2026.1` sources (Keystone, keystonemiddleware, keystoneauth, oslo.policy, Nova, Neutron, Cinder), not
captured lines; `<…>` marks a value.

Four different failures look alike from the outside:

1. **401, the token or the password is refused.** Keystone, or the `keystonemiddleware` in front of a service, does not
   accept the credentials.
2. **403, the token is accepted but the policy says no.** The user is known; the role or the scope does not allow the
   action.
3. **A service's own credentials are wrong.** Nova, Neutron or Cinder cannot get a token for its calls to another
   service; users see 500s or failed operations, not a 401.
4. **Keystone cannot be reached.** The middleware answers 503, the clients log connection errors.

How to read Keystone's own log and the middleware lines in detail is on
[reading the Keystone logs](reading-keystone-and-auth-logs.md).

## `The request you have made requires authentication. (HTTP 401) (Request-ID: req-<uuid>)`

- **Where:** printed by the `openstack` client or any keystoneauth user. The first sentence is Keystone's
  `Unauthorized` (`keystone/exception.py`); keystoneauth adds ` (HTTP 401)` and the request id
  (`HttpError` in `keystoneauth1/exceptions/http.py`). The `keystonemiddleware` in front of every other service
  answers a refused token with the same sentence (`auth_token/__init__.py`), so the text alone does not say who
  refused.
- **Means:** depends on who answered. The request id says which service: grep it in that service's log.
  - **Keystone answered** (the client was getting a token): wrong password, unknown user or project, a disabled user
    or project, a domain that does not match. Keystone logs `Authorization failed. <message> from <client address>`
    (WARNING).
  - **Another service answered** (the token was obtained, then refused): its `keystonemiddleware` asked Keystone to
    validate the token and Keystone said no. The service logs `Authorization failed for token` (WARNING) and
    `Rejecting request` (INFO).
- **Read next:** Keystone's log on every controller for the time of the request. A token refused by the services but
  issued by Keystone a moment before is the sign of Fernet keys that differ between controllers or of a clock offset.
- **Usual causes:** wrong credentials; Fernet keys not the same on all Keystone nodes (after a rotation that did not
  reach every node); a clock offset between controllers, so a fresh token looks expired; a token revoked (user
  disabled, password changed, role removed).
- **Fix:** [401 for tokens that should be valid: Fernet keys or a clock](../recovery/keystone-401-for-every-request-fernet-keys-or-clock.md);
  [expired certificates and Fernet keys](../diagnostics/expired-certificates-and-fernet-keys-break-every-api-call.md);
  [compute clock skew](../diagnostics/compute-clock-skew.md).

## 403: the policy refuses the action

The token is valid; the service's policy (oslo.policy) does not allow this user, role or scope to do this. Each
service words it differently:

| Message returned to the client | Service | Source |
|---|---|---|
| `Policy doesn't allow <rule> to be performed.` | Nova, Cinder | `PolicyNotAuthorized` in `nova/exception.py`, `cinder/exception.py` |
| `You are not authorized to perform the requested action: <rule>.` | Keystone | `ForbiddenAction` in `keystone/exception.py` |
| `<rule> is disallowed by policy` | Neutron (for a create, or a resource the user may see), and other services that pass oslo.policy's exception through | `PolicyNotAuthorized` in `oslo_policy/policy.py` |
| `<rule> requires a scope of [<scopes>], request was made with <scope> scope.` | any service enforcing scopes | `InvalidScope` in `oslo_policy/policy.py` |
| `The resource could not be found.` (404) | Neutron, when the user may not even see the resource | `policy_enforcement.py`: a refused action on a resource the user is not allowed to see is answered 404, not 403, so as not to reveal that it exists |

- **Where:** in the client's output. The service's access log has the request with status 403 (or 404 for Neutron);
  services usually do not log the policy decision at the default level.
- **Means:** the request reached the service with a valid token. The rule name in the message is the one to look up.
- **Read next:** the user's role assignments and the token's scope: `openstack role assignment list --user <user>
  --names`, then the service's policy for that rule (`oslopolicy-policy-generator --namespace <service>` prints the
  defaults; an override file in the service's configuration directory changes them).
- **Usual causes:** a missing role (`member`, `reader`, `admin`); a project-scoped token used for a system-scoped
  action, or the reverse; a policy override file left from an older release.
- **Fix:** no runbook yet; the change is a role assignment or a policy override.

## A service's own credentials: Nova towards Neutron, Glance, Cinder, and Neutron towards Nova

Services call each other with the user's token, or with their own service user (the `[keystone_authtoken]`,
`[neutron]`, `[nova]`, `[service_user]` sections of their configuration). When their own credentials are wrong, the
user sees a 500 or a failed operation, and the service logs the reason:

| Line | Where | Means | Next |
|---|---|---|---|
| `Neutron client was not able to generate a valid admin token, please verify Neutron admin credential located in nova.conf` (ERROR), then `Networking client is experiencing an unauthorized exception.` (`NeutronAdminCredentialConfigurationInvalid`) | `nova-api.log`, `nova-compute.log` (`nova/network/neutron.py`) | Nova's `[neutron]` credentials are refused by Keystone | the `[neutron]` section of `nova.conf`; Keystone's `Authorization failed` line for that user |
| `The [neutron] section of your nova configuration file must be configured for authentication with the networking service endpoint. …` (ERROR) | same | `[neutron] auth_type` is not set | the `[neutron]` section |
| `Not authorized for image <image-id>.` (`ImageNotAuthorized`) | `nova-compute.log`, the instance fault | Glance answered 401 or 403 to Nova for that image | the image's visibility and owner; Glance's log for the request |
| `Failed to notify nova on events: <events>` (ERROR, with a traceback naming `Unauthorized`) | `neutron-server.log` (`neutron/notifiers/nova.py`) | Neutron's `[nova]` credentials are refused: Nova never receives `network-vif-plugged`, and builds time out | the `[nova]` section of `neutron.conf`; see [instance stuck in BUILD waiting for vif-plugged](../lifecycle/instance-stuck-in-build-waiting-for-vif-plugged.md) |
| `Identity server rejected authorization` (INFO) | any service's log (`keystonemiddleware`) | the service's own `[keystone_authtoken]` user is refused, so it cannot validate any user token | `[keystone_authtoken]` of that service |

### Long operations: a user token that expires on the way

A user token can expire while Nova still uses it (a long boot from a large image, a live migration). When Nova sends a
service token with it (`[service_user] send_service_user_token`; the Nova default is `false`, Kolla-Ansible sets it to
`true`), the receiving middleware accepts the expired user token if the service token has one of the
`service_token_roles`. Without that, the receiving service logs `Invalid user token` (INFO) and answers 401 in the middle
of the operation. A middleware that receives a valid token in the service token header but without a service role logs
once: `A valid token was submitted as a service token, but it was not a valid service token. This is incorrect but
backwards compatible behaviour. This will be removed in future releases.` (WARNING).

## Keystone cannot be reached

| Line | Where | Means |
|---|---|---|
| `The Keystone service is temporarily unavailable.` (503 to the client), with `Unable to validate token: <error>` (CRITICAL) or `Bad response code while validating token: <status> <message>` (ERROR) in the service's log | any service with `keystonemiddleware` | the middleware could not get an answer from Keystone it trusts |
| `Unable to establish connection to <url>: <error>` | any keystoneauth user (clients, services) | TCP or DNS failure towards the URL: VIP down, HAProxy backend down, firewall |
| `Request to <url> timed out` | same | the connection hung: Keystone overloaded, or its database slow |
| `SSL exception connecting to <url>: <error>` | same | certificate expired, wrong CA bundle, wrong host name |
| `Failed to discover available identity versions when contacting <url>. Attempting to parse version from URL.` (WARNING) | same | the version discovery request to `auth_url` failed; often followed by one of the lines above |
| `Could not find requested endpoint in Service Catalog.` / `The service catalog is empty.` | same | the token has no endpoint for the service, interface or region asked: a wrong `region_name` or `valid_interfaces`, or a missing endpoint |

- **Read next:** HAProxy's log for the Keystone backend, then Keystone's own log and its database.
- **Fix:** [API returns 503: HAProxy backend down or VIP flapping](../recovery/api-returns-503-haproxy-backend-down-or-vip-flapping.md);
  [expired certificates](../diagnostics/expired-certificates-and-fernet-keys-break-every-api-call.md).

## A short reading order

1. The status: 401, 403 (or Neutron's 404), 500 or 503, and the request id in the client's output.
2. 401: grep the request id in the service that answered, then Keystone's log on every controller (`Authorization
   failed`), then the Fernet keys and the clocks.
3. 403: the rule name in the message, then the user's roles and the token's scope.
4. 500 on an operation between services: the service's log for `admin token`, `Unauthorized`, `Failed to notify nova`,
   then that service's credentials section.
5. 503: the middleware's `Unable to validate token`, then HAProxy and Keystone.

## Validity and limits

- Written from the `stable/2026.1` sources only. Every message is a format string from that source, not a captured line.
  The texts of other services' 401 and 403 answers (Glance, Heat, Octavia) are not all listed.
- Not run on a lab yet. Captures of a wrong password, a 403 policy refusal, wrong `[neutron]` credentials in Nova and an
  unreachable Keystone are requested.
- Policy defaults change between releases (secure RBAC, scopes); the rule names to look up are those of the running
  release.

---

*Provenance: written by agent:ozarland-author, from upstream source code (Keystone, keystonemiddleware, keystoneauth,
oslo.policy, Nova, Neutron, Cinder). Reviewed in its pull request (see the page history). Checked by a human operator:
not yet.*
