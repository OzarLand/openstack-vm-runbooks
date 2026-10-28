---
title: Requests fail with 401 for tokens that should be valid. Fernet keys differ between Keystone nodes, or a clock is off
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [keystone, nova, neutron]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers in the keystone group
  network: any
  storage: any
  os: any
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, multinode, 3 controllers (keystone x3, HAProxy), HTTP endpoints", date: 2026-10-27, note: "causes 1, 2 and 3 (clock behind with a revocation event, clock ahead by 2 days) injected on a disposable lab; Quick checks 1, 2, 3, the keystone_ssh repair (systemctl), the backups, fernet-push.sh from a chosen node and Verification run; kolla-ansible reconfigure -t keystone not run; cause 4 not set up on its own; chronyc makestep not run (systemd-timesyncd host, timedatectl set-ntp true used); no TLS backend" }
upstream_refs:
  - https://docs.openstack.org/keystone/2026.1/admin/fernet-token-faq.html
  - https://docs.openstack.org/keystone/2026.1/admin/tokens-overview.html
  - https://docs.openstack.org/keystonemiddleware/latest/middlewarearchitecture.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/shared-services/keystone-guide.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Requests fail with 401 for tokens that should be valid. Fernet keys differ between Keystone nodes, or a clock is off

Calls to Nova, Neutron or any other service are refused with 401, for some tokens or every other time, although
the user authenticated without trouble. The Keystone node asked to validate the token cannot read it, or finds it
expired or revoked. On Kolla-Ansible the two usual reasons are Fernet key sets that differ between Keystone nodes,
and a clock that is off on one of them.

They do not hit the same tokens. Keystone caches every token it issues in memcached, shared by all Keystone nodes,
for 10 minutes by default (see *Diagnosis*). So diverged keys break tokens *older than that*: long-running
clients, scripts that keep a token, Horizon sessions, and every token at once after a memcached restart. A fresh
token validates everywhere and hides the problem. A clock that is off breaks fresh tokens too.

## Symptom

The token request succeeds, a later service call fails:

```text
$ openstack token issue -f value -c expires
2026-10-27T10:12:44+0000
$ openstack server list
HttpException: 401: Client Error for url: https://<internal-vip-fqdn>:8774/v2.1/servers/detail, The request you have made requires authentication.
```

Commands that go through keystoneauth print `The request you have made requires authentication. (HTTP 401)
(Request-ID: req-<request-id>)`. With several Keystone nodes behind HAProxy, the failure is often intermittent: the
same command works, then fails, then works, depending on which node validated the token. With diverged keys, a
client that gets a new token (most CLI runs do) works for 10 minutes, then the same token fails on some nodes.
Intermittent is not the rule: on a three-controller lab, a token 11 minutes old from the node with the diverged key
got 401 on 8 Nova calls out of 8 through the VIP, in 3 seconds.

On the service side, `keystonemiddleware.auth_token` logs one warning per rejected token (in
`/var/log/kolla/nova/nova-api.log`, `/var/log/kolla/neutron/neutron-server.log` and so on):

```text
WARNING keystonemiddleware.auth_token [...] Authorization failed for token: …auth_token._exceptions.InvalidToken: Token authorization failed
```

That is the only line at the default log levels. The INFO lines that follow it in the source (`Identity response:
{"error": {"code": 404, "message": "Could not recognize Fernet token", ...}}` or `... "Failed to validate
token" ...`) are hidden by oslo.log's default `keystonemiddleware=WARN`, and Keystone logs a `404` on token
validation at DEBUG only. To see which of the two it is, use *Quick checks* 3. Keystone does log a request whose own
`X-Auth-Token` it cannot read (as in *Quick checks* 3), in `/var/log/kolla/keystone/keystone.log`:

```text
INFO keystone.server.flask.request_processing.middleware.auth_context [...] Invalid user token
WARNING keystone.server.flask.application [...] Authorization failed. The request you have made requires authentication. from <address>: keystone.exception.Unauthorized: [...]
```

## Impact

Every user and every service-to-service call that carries a token from the affected node fails, or fails every other
time. Automation and Horizon break; services that validate their peers' tokens (Nova calling Neutron, Cinder calling
Nova) fail in the middle of operations, which can leave instances or volumes in a transitional state. Running
instances are not affected.

## Quick checks

Read-only.

```bash
# 1. Can every node read every node's tokens? Run on each controller and compare the output.
sudo docker exec keystone_fernet bash -c 'cd /etc/keystone/fernet-keys && md5sum *'
#    Key 0 is the staged key, the highest number is the primary key that encrypts new tokens.
#    The deciding test: is each node's primary key (its highest number) present, by checksum, in every other
#    node's set, under any number, key 0 included? If yes, every node decrypts every token, even if the lists differ.
#    Different lists alone are not a fault: after one rotation whose push failed they differ, and nothing breaks.
#    This is the check that decides cause 1. Check 3 cannot, with a fresh token (see below).

# 2. Is the key distribution working? keystone_fernet's health check runs fernet-node-sync.sh --check (key 0
#    present and owned by keystone) and fernet-push.sh --check (an rsync dry run to the other nodes).
sudo docker ps -a --filter name=keystone --format '{{.Names}} {{.Status}}'
#    keystone_fernet "(unhealthy)": key 0 missing or with the wrong owner, or the push to the other nodes fails
#    (keystone_ssh down, SSH key, network). A node whose keystone_ssh is down stays healthy itself: its peers,
#    which cannot push to it, turn unhealthy (on the lab, within 90 s of the stop). Without -a, a stopped
#    keystone_ssh is not listed at all.

# 3. Which node rejects the token? Validate one token against each Keystone backend directly, bypassing HAProxy
#    (needs admin credentials in the environment). The token must be older than the token cache TTL
#    ([cache] expiration_time, 600 s by default): a younger one comes from the shared cache on every node and is
#    never decrypted, so this check returns 200 everywhere even with diverged keys.
TOKEN=$(openstack token issue -f value -c id); sleep 660
for n in <controller-1-api-address> <controller-2-api-address> <controller-3-api-address>; do
  echo "$n $(curl -s -o /dev/null -w '%{http_code}' -H "X-Auth-Token: $TOKEN" -H "X-Subject-Token: $TOKEN" \
    http://$n:5000/v3/auth/tokens)"
done
#    expected: 200 everywhere. A 401 on one node is the problem: the token is also the caller (X-Auth-Token), so a
#    node that cannot read it refuses the request itself, "The request you have made requires authentication".
#    To get the 404 and its reason, send a fresh admin token as X-Auth-Token and keep the old one as
#    X-Subject-Token (drop -o /dev/null to see the body): "Could not recognize Fernet token" (that node does not
#    have the key) or "Failed to validate token" (expired or revoked, as seen by that node's clock).
#    Only the first answer for a token is reliable: a node that validates it writes it back to the shared cache
#    for another 600 s, and every node asked after it answers 200 from the cache. On the lab, this loop as written
#    gave 200 on the three nodes with diverged keys. Put the node you suspect first, or repeat the check with
#    another aged token and another node first. Use https:// if kolla_enable_tls_backend is set.

# 4. Clocks of the Keystone nodes
chronyc tracking | grep -e "System time" -e "Leap status"
#    Hosts that use systemd-timesyncd (the Ubuntu default) have no chronyc:
timedatectl show -p NTPSynchronized --value     # yes / no
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

### Why a fresh token validates everywhere

`[token] caching` and `[token] cache_on_issue` both default to `True` (`keystone/conf/token.py`), and Kolla-Ansible
enables the global cache with `oslo_cache.memcache_pool` on every memcached host (`[cache]` in `keystone.conf.j2`), so
all Keystone nodes share one token cache. `_validate_token`, which decrypts the token, is memoized in it
(`keystone/token/provider.py`); with `[token] cache_time` unset, an entry lives `[cache] expiration_time`, 600 s by
default (oslo.cache). Keystone writes the entry when it issues the token. So for 10 minutes, any node answers from
the cache, whether or not it has the key.

The expiry check in `_is_valid_token` is not cached: each node compares the token's `expires_at` with its own clock on
each validation. The revocation check (`check_revocation_v3`) is memoized too, and compares the token's `issued_at`,
set by the issuing node's clock, with the events' `issued_before`, set by the clock of the node that wrote them. A
clock that is off therefore shows on fresh tokens, through the issuing node or the validating node.

### How Kolla-Ansible keeps the keys in sync

- The `keystone_fernet` container on each Keystone node runs cron, with rotation slots computed by
  `fernet_rotate_cron_generator.py` from `fernet_key_rotation_interval` (by default `fernet_token_expiry +
  fernet_token_allow_expired_window` = 86400 + 172800 seconds, 4320 minutes) and the node's index in the `keystone`
  group. With these defaults and three nodes, running the generator gives: the first node rotates on Sundays at
  00:00, the second on Wednesdays at 00:00, and the third has no slot at all (its `crontab` has only the `PATH` line). Check yours
  in `/etc/kolla/keystone-fernet/crontab` on each node.
- At its turn, a node runs `keystone-manage fernet_rotate`, then `fernet-push.sh`, which copies its key directory to
  every other Keystone node with `rsync --delete` over SSH to their `keystone_ssh` container.
- `kolla-ansible deploy` and `reconfigure` run `fernet-push.sh` once, from the first node of the `keystone` group
  where `keystone_fernet` runs (`distribute_fernet.yml`).
- `max_active_keys` is computed so that every key still needed for a valid token is kept:
  `((fernet_token_expiry + fernet_token_allow_expired_window + fernet_key_rotation_interval - 1) // fernet_key_rotation_interval) + 2`.

A rotation (`rotate_keys` in `keystone/common/fernet_utils.py`) promotes the staged key `0` to be the new primary and
creates a new `0`. Key `0` came from the last push, so it is the same on every node: after one rotation, the new
primary is already in every other node's set, as their key `0`. Keystone decrypts with every key in the repository
(`MultiFernet` in `keystone/token/token_formatters.py`), and the `rotate_keys` docstring says this lets you rotate on
one node in a cluster before syncing. So **one missed push is harmless by design**: the lists differ, every token
still validates everywhere.

Tokens break only when a node's primary is missing from another node's set:

- a second rotation on the same node without a push in between: its new primary is its own new `0`, which no other
  node has. With the default `max_active_keys = 3`, that rotation also purges the key that is still the other
  nodes' primary (`Excess key to purge: /etc/keystone/fernet-keys/1` in its log), so the
  node rejects their tokens too: the failure goes both ways (seen on the lab);
- rotations on two nodes without pushing, where at least one of them rotates twice (two nodes that each rotate once
  from the same pushed set promote the same key `0`, so they share a primary);
- keys created independently: a node rebuilt with `fernet_setup`, a key volume restored from a backup, keys copied by
  hand.

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
| A node's primary key is missing from another node's set (check 1); tokens older than 10 minutes get 404 `Could not recognize Fernet token` on some nodes | Cause 1: Fernet key sets diverged |
| Key lists differ, but every node's primary is in every other set | Not a fault: one rotation not yet pushed. Look for another cause, and repair the push (cause 2) before the next rotation |
| `keystone_fernet` unhealthy on one node | Cause 2: the push from that node fails, or its key 0 is missing. It becomes cause 1 at that node's next rotation |
| Keys identical, 404 `Failed to validate token` for fresh tokens issued by one node, for one user or project after a change on it | Cause 3: clock behind on the issuing node |
| Keys identical, every token rejected by one node as `Failed to validate token` | Cause 3: clock ahead on the validating node by more than the token lifetime |
| Every node answers 404 `Could not recognize Fernet token` for every token | Cause 4: a token issued by a key no node has any more (keys replaced on every node) |

### Cause 1: Fernet key sets differ

*Quick checks* 1 shows a node whose primary key is in no other node's set (under any number). Tokens it issued are
rejected by the others once they leave the token cache. Different lists where every primary is still present
everywhere are not this cause.

### Cause 2: the key push fails

`keystone_fernet` is unhealthy, or `keystone_ssh` is not running on a peer. Tell the two sides apart:

- a node that cannot **send** (its `fernet-push.sh` fails: its SSH key, the network, the peers' `keystone_ssh`):
  its rotations stay local. One is harmless; the second makes cause 1;
- a node that cannot **receive** (its own `keystone_ssh` is down): the other nodes' pushes to it fail, but a push
  from a peer that does reach it still overwrites its set with `rsync --delete`. Its own rotations, pushed to the
  others, can wipe keys they rotated in the meantime.

### Cause 3: clock skew on a Keystone node

*Quick checks* 4 shows a node whose time differs from the others, or chrony not synchronised (`Leap status : Not
synchronised`).

### Cause 4: keys replaced everywhere

All key sets are identical, but none decrypts the tokens clients hold: keys regenerated with `fernet_setup` on every
node, or a key volume restored from an old backup. Every client must authenticate again; services do it by themselves
at the next 401.

## Resolution

### For causes 1 and 2: make every node use one key set

Do this only when *Quick checks* 1 shows a primary key missing from another node's set. If every primary is
present everywhere, no push is needed for the 401s: repair the push path (cause 2) and look at the clock (cause 3).

**Choose the source node.** Compare *Quick checks* 1 across nodes, and the rotation lines in each node's
`keystone.log`. Do not go by the highest key number alone: once the sets have diverged (a node that rotated twice
without pushing, a rebuilt node), two nodes can each have a key `3` with different contents. Take the node whose set
the others are closest to and that rotated last, usually the one most clients have been getting tokens from. Its
primary key is the one you keep.

**Repair the push path** if it is broken (cause 2):

> ⚠️ **State-changing.** Starts the SSH endpoint that receives the keys on that node. Undo: `sudo systemctl stop
> kolla-keystone_ssh-container.service`.

```bash
sudo systemctl start kolla-keystone_ssh-container.service    # on each peer where it is stopped
```

On Kolla-Ansible 2026.1 each container runs under a systemd unit. `sudo docker start keystone_ssh` also starts the
container, but leaves `kolla-keystone_ssh-container.service` inactive (seen on the lab); `systemctl start` then
attaches the unit to the running container. Then check that `keystone_fernet` becomes healthy on the source node (10
seconds on the lab).

**Back up every node's keys**, into a directory only root can read:

```bash
# on each Keystone node
sudo install -d -m 700 /root/fernet-backup
sudo sh -c 'umask 077; docker exec keystone_fernet tar -C /etc/keystone -czf - fernet-keys > /root/fernet-backup/fernet-keys-$(hostname)-$(date +%F).tgz'
```

The archives hold secret keys: delete them once the incident is closed.

**Push from the source node.**

> 🛑 **Destructive.** `fernet-push.sh` copies the source node's key directory to every other Keystone node with
> `rsync --delete`. What is lost: the other nodes' own primary keys, and with them every live token those nodes
> issued and that is no longer in the cache, up to two thirds of all tokens with three nodes. Who is affected:
> users and scripts get a 401 and must authenticate again; services re-authenticate their own tokens, but not the
> user tokens they forward (Nova to Neutron, Cinder or Glance), so builds, attachments and migrations in progress
> can fail. Precondition: the source chosen by checksum and rotation time as above; a quiet window with no
> provisioning in progress; outside the rotation slots (`/etc/kolla/keystone-fernet/crontab`); the backups taken.
> There is no real undo: restoring the archives brings the divergence back.

```bash
# on the source node
sudo docker exec keystone_fernet /usr/bin/fernet-push.sh
```

`kolla-ansible reconfigure -i <inventory> -t keystone` also pushes, but from the first node of the `keystone` group
where `keystone_fernet` runs, not from the node you chose.

> 🛑 **Destructive.** Same loss as above, from a source you do not choose: if that node has the wrong set, it
> replaces the good one everywhere. It also re-renders the Keystone configuration and runs the Keystone bootstrap,
> so it can restart the `keystone`, `keystone_fernet` and `keystone_ssh` containers, with failed requests while
> they restart. Use it only when *Quick checks* 1 shows that the first node of the group is the right source. Same
> preconditions; no real undo.

```bash
# not run on the lab
kolla-ansible reconfigure -i <inventory> -t keystone
```

Keystone loads the key repository each time it encrypts or decrypts a token (the `crypto` property of
`TokenFormatter`), so no restart is needed. Run *Quick checks* 1 again, then 3 with a token older than 10 minutes.

### For cause 3: fix time synchronisation

Fix chrony (or your time service) on the node that is off, as for any host: see
[the clock of one compute has drifted](../diagnostics/compute-clock-skew.md) for how to read `chronyc` and correct a
skew.

> ⚠️ **State-changing.** Steps the system clock of the node at once instead of slewing it. Services on that node
> see time jump; log lines from before and after do not sort. Precondition: the node's time sources are reachable
> (`chronyc sources` shows `^*` or `^+`). Undo: none needed.

```bash
sudo chronyc makestep        # not run on the lab (no chrony there; the timesyncd form below was)
```

On a host with systemd-timesyncd instead of chrony (no `chronyc`), turn NTP back on; timesyncd steps the clock at
its first synchronisation (on the lab, 2 seconds after the command, for a clock 300 seconds behind):

> ⚠️ **State-changing.** Enables and starts systemd-timesyncd, which steps the clock at its first synchronisation:
> services on that node see time jump. Precondition: the NTP server is reachable and no other time daemon runs.
> Undo: `sudo timedatectl set-ntp false`.

```bash
sudo timedatectl set-ntp true
timedatectl show -p NTPSynchronized --value     # yes
```

Tokens issued by that node while it was behind stay rejected after a revocation event until they expire (on the lab,
still `404 Failed to validate token` on every node after the fix); users who hit it get a new token by
authenticating again.

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

# A token older than the cache TTL validates on every node (Quick checks 3, with the sleep): 200 from each
# And through the VIP, several times in a row
for i in 1 2 3 4 5 6; do openstack server list --limit 1 -f value -c ID >/dev/null && echo ok; done
```

The `Authorization failed for token` warnings stop in the service logs, apart from clients that still hold old
tokens and authenticate again.

## Lab observations

On Kolla-Ansible 22.2.0 (2026.1), three controllers behind HAProxy, Ubuntu 24.04, HTTP internal endpoints:

- **Causes 2 then 1**: `keystone_ssh` stopped on controller-1 (`systemctl stop`), then Kolla's `fernet-rotate.sh`
  run twice by hand on controller-3; both pushes failed (`ssh: connect to host <controller-1> port 8023: Connection
  refused`). After the first rotation the lists differed but every token validated everywhere, also 10 minutes
  later. After the second, controller-3 held `{0, 2, 3}` and the others `{0, 1}`. A fresh token from controller-3
  validated on the three nodes (cache); 618 seconds later, *Quick checks* 3 gave 401 on controller-1 and -2 and 200
  on controller-3, and a controller-1 token got 401 on controller-3. With a fresh caller token the 404 read `Could
  not recognize Fernet token`. Nova through the VIP: 401 on 8 calls out of 8.
- `keystone_fernet` turned unhealthy on controller-2 and -3 (the senders), not on controller-1.
- **Resolution**: `keystone_ssh` back, keys backed up, `fernet-push.sh` from controller-3 (rotated last): identical
  sets at once, `keystone_fernet` healthy, *Verification* passed (6 CLI runs out of 6 through the VIP, tokens
  issued after the push 200 on every node after 10 minutes). A controller-1 token issued 24 seconds before the push
  got `404 Could not recognize Fernet token` on every node once out of the cache: the loss the push warning
  describes.
- **Cause 3**: controller-2 set 300 seconds behind, then the test user's password changed through controller-1: a
  fresh token for that user from controller-2 got `404 Failed to validate token` on the three nodes, one from
  controller-1 got 200. Then controller-2 set 2 days ahead: it refused every token (401 when the token is also the
  caller), the other two accepted them.
- Not run: `kolla-ansible reconfigure -i <inventory> -t keystone` (the second push method); cause 4 was not set up
  on its own (only seen on the token issued before the push); `chronyc makestep` (no chrony on the lab hosts:
  `timedatectl set-ntp true` instead); TLS on the backends (`kolla_enable_tls_backend`).

## Validity and limits

- Written from Keystone 2026.1 (`keystone/token/token_formatters.py`, `keystone/token/providers/fernet/core.py`,
  `keystone/token/provider.py`, `keystone/revoke/backends/sql.py`, `keystone/common/fernet_utils.py`,
  `keystone/server/flask/application.py` for what Keystone logs), keystonemiddleware 12.0.0 (the 2026.1
  upper constraint: `auth_token/__init__.py` and `_identity.py`), oslo.log `DEFAULT_LOG_LEVELS`, openstacksdk and
  keystoneauth1 for the client messages, and Kolla-Ansible 2026.1 (the `keystone` role: `keystone.conf.j2`,
  `fernet-push.sh.j2`, `fernet-rotate.sh.j2`, `fernet-healthcheck.sh.j2`, `distribute_fernet.yml`, `config.yml`, and
  `group_vars/all/keystone.yml`, `files/fernet_rotate_cron_generator.py`, `fernet-node-sync.sh.j2`), `keystone/conf/token.py`
  and oslo.cache 4.1.1 for the token cache. Causes 1, 2 and 3 and the push from a chosen node reproduced on a
  three-controller Kolla-Ansible 22.2.0 lab (see *Lab observations*); `kolla-ansible reconfigure`, TLS backends and
  cause 4 on its own were not.
- Kolla-Ansible only. Other deployment tools distribute Fernet keys their own way (shared storage, configuration
  management); the diagnosis applies, the commands and container names do not.
- The validation call of *Quick checks* 3 uses an admin token as both the caller and the subject token, which the
  default policy allows (a user may validate their own token).
- The rotation slots were computed by running Kolla-Ansible 2026.1's `fernet_rotate_cron_generator.py` with the
  default interval (`-t 4320 -n 3`). Another interval or number of nodes gives other slots.
- Not covered: tokens from federation, application credentials and trusts beyond what the revocation check does,
  memcached failures and slow token validation (a separate page), and the `credential` key repository, which
  Kolla-Ansible distributes differently.

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
