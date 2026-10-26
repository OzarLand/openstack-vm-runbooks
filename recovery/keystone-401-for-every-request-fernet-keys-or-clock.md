---
title: Requests fail with 401 although the token was just issued. Fernet keys differ between Keystone nodes, or a clock is off
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [keystone, nova, neutron]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers in the keystone group
  network: any
  storage: any
  os: any
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/keystone/2026.1/admin/fernet-token-faq.html
  - https://docs.openstack.org/keystone/2026.1/admin/tokens-overview.html
  - https://docs.openstack.org/keystonemiddleware/latest/middlewarearchitecture.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/shared-services/keystone-guide.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Requests fail with 401 although the token was just issued. Fernet keys differ between Keystone nodes, or a clock is off

A user authenticates without trouble, then the next call to Nova, Neutron or any other service is refused with 401,
always or every other time. The password is right, the token is fresh: the problem is that the Keystone node asked
to validate the token cannot read it, or finds it expired or revoked. On Kolla-Ansible the two usual reasons are
Fernet key sets that differ between Keystone nodes, and a clock that is off on one of them.

## Symptom

The token request succeeds, the service call fails:

```text
$ openstack token issue -f value -c expires
2026-10-27T10:12:44+0000
$ openstack server list
HttpException: 401: Client Error for url: https://<internal-vip-fqdn>:8774/v2.1/servers/detail, The request you have made requires authentication.
```

Commands that go through keystoneauth print `The request you have made requires authentication. (HTTP 401)
(Request-ID: req-<request-id>)`. With several Keystone nodes behind HAProxy, the failure is often intermittent: the
same command works, then fails, then works, depending on which node validated the token.

On the service side, `keystonemiddleware.auth_token` logs one warning per rejected token (in
`/var/log/kolla/nova/nova-api.log`, `/var/log/kolla/neutron/neutron-server.log` and so on):

```text
WARNING keystonemiddleware.auth_token [...] Authorization failed for token
```

That is the only line at the default log levels. The INFO lines that follow it in the source (`Identity response:
{"error": {"code": 404, "message": "Could not recognize Fernet token", ...}}` or `... "Failed to validate
token" ...`) are hidden by oslo.log's default `keystonemiddleware=WARN`, and Keystone logs a `404` on token
validation at DEBUG only. To see which of the two it is, use *Quick checks* 3.

## Impact

Every user and every service-to-service call that carries a token from the affected node fails, or fails every other
time. Automation and Horizon break; services that validate their peers' tokens (Nova calling Neutron, Cinder calling
Nova) fail in the middle of operations, which can leave instances or volumes in a transitional state. Running
instances are not affected.

## Quick checks

Read-only.

```bash
# 1. Are the Fernet keys the same on every Keystone node? Run on each controller and compare the output.
sudo docker exec keystone_fernet bash -c 'cd /etc/keystone/fernet-keys && ls && md5sum *'
#    expected: the same file names (0, 1, 2, ...) and the same checksums on every node.
#    Key 0 is the staged key, the highest number is the primary key that encrypts new tokens.

# 2. Is the key distribution working? keystone_fernet's health check is an rsync dry run to the other nodes.
sudo docker ps --filter name=keystone --format '{{.Names}} {{.Status}}'
#    keystone_fernet "(unhealthy)" means the push to the other nodes fails (keystone_ssh down, SSH key, network).

# 3. Which node rejects the token? Issue one token, then validate it against each Keystone backend directly,
#    bypassing HAProxy (needs admin credentials in the environment).
TOKEN=$(openstack token issue -f value -c id)
for n in <controller-1-api-address> <controller-2-api-address> <controller-3-api-address>; do
  echo "$n $(curl -s -o /dev/null -w '%{http_code}' -H "X-Auth-Token: $TOKEN" -H "X-Subject-Token: $TOKEN" \
    http://$n:5000/v3/auth/tokens)"
done
#    expected: 200 everywhere. A 404 on one node is the problem. The body says why:
#    "Could not recognize Fernet token" (that node does not have the key) or "Failed to validate token"
#    (expired or revoked, as seen by that node's clock). Use https:// if kolla_enable_tls_backend is set.

# 4. Clocks of the Keystone nodes
chronyc tracking | grep -e "System time" -e "Leap status"
date -u +%FT%TZ
```

## Diagnosis

### How a Fernet token is validated

A Fernet token is encrypted by the primary key of the Keystone node that issued it. Any node can validate it if it
has that key in its key repository (`/etc/keystone/fernet-keys`, `[fernet_tokens] key_repository`). If none of its
keys decrypts the token, Keystone answers `404` with `Could not recognize Fernet token` (`keystone/token/token_formatters.py`,
re-raised as `TokenNotFound` in `keystone/token/providers/fernet/core.py`). If it decrypts but the token's
`expires_at` is in the past for that node's clock, or a revocation event matches it, the answer is `404` with
`Failed to validate token` (`keystone/token/provider.py`). Keystonemiddleware turns both into a `401` for the client.

### How Kolla-Ansible keeps the keys in sync

- The `keystone_fernet` container on each Keystone node runs cron. Each node gets its own rotation time
  (`fernet_rotate_cron_generator.py` spreads them over `fernet_key_rotation_interval`, by default
  `fernet_token_expiry + fernet_token_allow_expired_window` = 86400 + 172800 seconds).
- At its turn, a node runs `keystone-manage fernet_rotate`, then `fernet-push.sh`, which copies its key directory to
  every other Keystone node with `rsync --delete` over SSH to their `keystone_ssh` container.
- `kolla-ansible deploy` and `reconfigure` run `fernet-push.sh` once, from the first node of the `keystone` group
  where `keystone_fernet` runs (`distribute_fernet.yml`).
- `max_active_keys` is computed so that every key still needed for a valid token is kept:
  `((fernet_token_expiry + fernet_token_allow_expired_window + fernet_key_rotation_interval - 1) // fernet_key_rotation_interval) + 2`.

So the key sets diverge when a rotation happened on one node and its push failed, or when a node got keys that the
others do not have (rebuilt node, restored volume, keys copied by hand).

A rotation leaves these lines in `/var/log/kolla/keystone/keystone.log` of the node that rotated (logger
`keystone.common.fernet_utils`):

```text
INFO keystone.common.fernet_utils [...] Starting key rotation with 3 key files: [...]
INFO keystone.common.fernet_utils [...] Created a new temporary key: /etc/keystone/fernet-keys/0.tmp
INFO keystone.common.fernet_utils [...] Current primary key is: 2
INFO keystone.common.fernet_utils [...] Next primary key will be: 3
INFO keystone.common.fernet_utils [...] Promoted key 0 to be the primary: 3
INFO keystone.common.fernet_utils [...] Become a valid new key: /etc/keystone/fernet-keys/0
```

The time of the last rotation, compared with the time the 401s started, is the strongest clue.

### What a clock off does

Keystone compares the token's `expires_at` with its own clock, and compares the token's `issued_at` with the
`issued_before` of revocation events (`keystone/revoke/backends/sql.py`). Revocation events are written when a token
is revoked, a password changes, a role assignment is removed, a project or user is disabled. A node whose clock is
behind issues tokens that look older than they are:

- after a revocation event for a user or project, that node's new tokens fall before `issued_before` and are
  rejected as revoked by every node, until the real time passes the skew;
- a node whose clock is ahead by more than the token lifetime (`fernet_token_expiry`, 86400 seconds by default)
  sees every token as expired.

The first case is the common one: a small skew of a few minutes makes a user's fresh tokens fail right after a
password change or a role change, and only for tokens from one node.

## Possible causes

| What you see | Cause |
|---|---|
| Checksums or key file names differ between nodes; validation 404 `Could not recognize Fernet token` on some nodes | Cause 1: Fernet key sets diverged |
| `keystone_fernet` unhealthy on one node | Cause 1, through cause 2: the push from that node fails |
| Keys identical, 404 `Failed to validate token` on one node, for one user or project after a change on it | Cause 3: clock behind on the issuing node |
| Keys identical, every token rejected by one node as `Failed to validate token` | Cause 3: clock ahead on the validating node by more than the token lifetime |
| Every node answers 404 `Could not recognize Fernet token` for every token | Cause 4: a token issued by a key no node has any more (keys replaced on every node) |

### Cause 1: Fernet key sets differ

*Quick checks* 1 shows different files or checksums. Tokens issued by a node whose primary key the others do not have
are rejected by those others.

### Cause 2: the key push fails

`keystone_fernet` is unhealthy, or `keystone_ssh` is not running on a peer. Rotations still happen locally on that
node, so its set drifts further at each turn.

### Cause 3: clock skew on a Keystone node

*Quick checks* 4 shows a node whose time differs from the others, or chrony not synchronised (`Leap status : Not
synchronised`).

### Cause 4: keys replaced everywhere

All key sets are identical, but none decrypts the tokens clients hold: keys regenerated with `fernet_setup` on every
node, or a key volume restored from an old backup. Every client must authenticate again; services do it by themselves
at the next 401.

## Resolution

### For causes 1 and 2: make every node use one key set

Choose the node to copy from: the one whose primary key (the highest number) is the newest, usually the one that
rotated last according to `keystone.log`. Its set is the one that encrypts current tokens from that node.

First repair the push path if it is broken (cause 2): start the `keystone_ssh` container on the peers
(`sudo docker start keystone_ssh`) and check that `keystone_fernet` becomes healthy.

> ⚠️ **State-changing.** Copies the chosen node's Fernet keys to every other Keystone node with `rsync --delete`.
> Keys present only on the other nodes are removed: tokens they encrypted become invalid, and their holders get one
> 401 and authenticate again (services do it by themselves). Precondition: the push path works (`keystone_fernet`
> healthy on the chosen node). Undo: none; the removed keys cannot be brought back unless you saved them.

Save every node's keys first, then push from the chosen node:

```bash
# on each Keystone node
sudo docker exec keystone_fernet tar -C /etc/keystone -czf - fernet-keys > fernet-keys-$(hostname)-$(date +%F).tgz
chmod 600 fernet-keys-*.tgz

# on the chosen node
sudo docker exec keystone_fernet /usr/bin/fernet-push.sh
```

The archives hold secret keys: keep them only as long as needed, then delete them.

Alternatively, `kolla-ansible reconfigure -i <inventory> -t keystone` pushes from the first node of the `keystone`
group where `keystone_fernet` runs, which is not necessarily the node you chose.

Keystone loads the key repository each time it encrypts or decrypts a token (the `crypto` property of
`TokenFormatter`), so no restart is needed. Run *Quick checks* 1 and 3
again.

### For cause 3: fix time synchronisation

Fix chrony (or your time service) on the node that is off, as for any host: see
[the clock of one compute has drifted](../diagnostics/compute-clock-skew.md) for how to read `chronyc` and correct a
skew.

> ⚠️ **State-changing.** Steps the system clock of the node at once instead of slewing it. Services on that node
> see time jump; log lines from before and after do not sort. Precondition: the node's time sources are reachable
> (`chronyc sources` shows `^*` or `^+`). Undo: none needed.

```bash
sudo chronyc makestep
```

Tokens issued by that node while it was behind stay rejected after a revocation event until they expire or real time
catches up; users who hit it get a new token by authenticating again.

### For cause 4: nothing to fix on Keystone

Clients holding old tokens must authenticate again. Services do it on their own after a 401; long-running clients
with a cached token (scripts, CI) must be restarted or re-authenticated. Make sure every node has the same set
(*Quick checks* 1) so it does not happen again at the next request.

## Verification

```bash
# Same key set on every node
sudo docker exec keystone_fernet bash -c 'cd /etc/keystone/fernet-keys && md5sum *'   # on each node, compare

# keystone_fernet healthy on every node
sudo docker ps --filter name=keystone_fernet --format '{{.Names}} {{.Status}}'

# A fresh token validates on every node (Quick checks 3): 200 from each
# And through the VIP, several times in a row
for i in 1 2 3 4 5 6; do openstack server list --limit 1 -f value -c ID >/dev/null && echo ok; done
```

The `Authorization failed for token` warnings stop in the service logs, apart from clients that still hold old
tokens and authenticate again.

## Validity and limits

- Written from Keystone 2026.1 (`keystone/token/token_formatters.py`, `keystone/token/providers/fernet/core.py`,
  `keystone/token/provider.py`, `keystone/revoke/backends/sql.py`, `keystone/common/fernet_utils.py`,
  `keystone/server/flask/application.py` for what Keystone logs), keystonemiddleware 12.0.0 (the 2026.1
  upper constraint: `auth_token/__init__.py` and `_identity.py`), oslo.log `DEFAULT_LOG_LEVELS`, openstacksdk and
  keystoneauth1 for the client messages, and Kolla-Ansible 2026.1 (the `keystone` role: `keystone.conf.j2`,
  `fernet-push.sh.j2`, `fernet-rotate.sh.j2`, `fernet-healthcheck.sh.j2`, `distribute_fernet.yml`, `config.yml`, and
  `group_vars/all/keystone.yml`). Not reproduced on a lab.
- Kolla-Ansible only. Other deployment tools distribute Fernet keys their own way (shared storage, configuration
  management); the diagnosis applies, the commands and container names do not.
- The validation call of *Quick checks* 3 uses an admin token as both the caller and the subject token, which the
  default policy allows (a user may validate their own token).
- Not covered: tokens from federation, application credentials and trusts beyond what the revocation check does,
  `[token] cache_on_issue`, memcached failures and slow token validation (a separate page), and the `credential`
  key repository, which Kolla-Ansible distributes differently.

## Related

- [Every API call fails at once: expired certificates or Fernet keys](../diagnostics/expired-certificates-and-fernet-keys-break-every-api-call.md):
  the read-only overview, including certificates.
- [The clock of one compute has drifted](../diagnostics/compute-clock-skew.md)
- Keystone 2026.1: [Fernet token FAQ](https://docs.openstack.org/keystone/2026.1/admin/fernet-token-faq.html)

---

*Provenance: written by agent:ozarland-author, from the Keystone, keystonemiddleware and Kolla-Ansible 2026.1 source
and documentation. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/openstack-vm-runbooks/issues/new?template=documentation-proposal.yml&page=recovery/keystone-401-for-every-request-fernet-keys-or-clock.md&title=Feedback%3A+recovery/keystone-401-for-every-request-fernet-keys-or-clock.md):
your release, your deployment method, and what happened.
