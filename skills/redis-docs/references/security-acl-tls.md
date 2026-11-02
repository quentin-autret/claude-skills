# Security: network exposure, ACL users and TLS

Defaults that leave Redis open, ACL rule semantics and traps, the Redis 8 module categories, TLS settings and recent security fixes for Redis Open Source 8.10.

## Contents

- Defaults and exposure
- ACL users
- ACL rule traps
- Key and channel permissions
- ACL files and rotation
- TLS
- Security fixes by release
- Old patterns

## Defaults and exposure

- The `default` user is `on nopass ~* &* +@all`: every new connection is authenticated with full rights until you change it. Default Redis also listens on all interfaces.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/ and https://redis.io/docs/latest/operate/oss_and_stack/management/admin/
- Protected mode (on by default) only lets loopback clients in while the default user has no password. Set a password or ACL users rather than disabling it.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/ and https://redis.io/docs/latest/develop/reference/errors/common/denied/
- Only trusted clients on trusted networks reach Redis. Untrusted users go through an application layer. Firewall the port, and `bind 127.0.0.1` when possible.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/
- `CONFIG` lets a client change the dump directory and file name and so write RDB files to arbitrary paths as the Redis user. Deny it to application users.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/
- `SORT` uses a non-randomized qsort, so crafted input can force the quadratic worst case.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/

## ACL users

- ACL users (6.0) are the recommended authentication. `requirepass` is legacy and now just sets the `default` user's password, and `AUTH <password>` alone means user `default`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/ and https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/
- Give each app or worker its own least-privilege user. Default rule shape: categories plus exclusions.

```
ACL SETUSER app on >S3cretFromGENPASS ~app:* &app:* +@all -@dangerous -@admin
```
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/
- Generate passwords with `ACL GENPASS`. ACL passwords are hashed with SHA256 for fast AUTH, so their strength must come from the secret itself.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/
- Check category contents with `ACL CAT <category>`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/

## ACL rule traps

- A new user is `off`, passwordless, with no commands, keys or channels. A user that is neither `nopass` nor given a password can never log in.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/
- `ACL SETUSER` is cumulative and applies rules left to right: `+set` then `+get` grants both. Start with `reset` to redefine a user from scratch.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/ and https://redis.io/docs/latest/commands/acl-setuser/
- `nopass` accepts ANY password and wipes the configured ones.
  https://redis.io/docs/latest/commands/acl-setuser/
- `ACL SETUSER user off` does not disconnect live sessions. Use `CLIENT KILL` with the user filter, or `ACL DELUSER`.
  https://redis.io/docs/latest/commands/acl-setuser/
- Key patterns do not limit keyless commands: `~tenant1:* +@all` can still `FLUSHALL`. Remove them explicitly:

```
ACL SETUSER tenant1 on >strong-password ~tenant1:* +@all -flushall -flushdb -swapdb
```
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/
- `+@all` grants every command, including commands loaded later via modules, so `+@all -...` needs care. The ACL page still says categories other than `+@all` never include module commands, but since Redis 8 the standard categories (`@read`, `@write`) cover the built-in JSON, time series, vector and probabilistic commands (next item).
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/ and https://redis.io/docs/latest/develop/whats-new/8-0/
- Redis 8 moved module commands into the standard categories and added module categories (json, search, timeseries, bloom, cuckoo, cms, topk, tdigest). A user with `+@all -@write` can no longer run `JSON.SET`. Review existing ACLs on upgrade.
  https://redis.io/docs/latest/develop/whats-new/8-0/ and https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/
- Search additionally needs the user's key patterns to cover the index prefixes.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/

## Key and channel permissions

- Read permission covers returned data only, not metadata: `LPUSH` needs only write. To hide a key, grant nothing on it.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/
- `%R~` and `%W~` (7.0) split read and write patterns, for example `COPY` from `app2:*` into `app1:*`: `+@all ~app1:* %R~app2:*`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/
- Selectors `(...)` add alternative rule sets but cannot be edited, only cleared with `clearselectors`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/
- Channels: `PUBLISH`/`SUBSCRIBE` are glob matched, `PSUBSCRIBE` needs a literal pattern match. `acl-pubsub-default` is `resetchannels` since 7.0.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/
- Replicas and Sentinel get only the commands they need:

```
ACL SETUSER replica-user on >somepassword +psync +replconf +ping
```
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/

## ACL files and rotation

- With several users, keep them in `aclfile /etc/redis/users.acl`. `ACL LOAD` is all-or-nothing and `ACL SAVE` writes the file. Users in `redis.conf` and an ACL file are mutually exclusive, and `CONFIG REWRITE` does not run `ACL SAVE`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/
- Store `#<sha256>` instead of cleartext. Rotate with two active passwords: `>new`, later `<old`.
  https://redis.io/docs/latest/commands/acl-setuser/
- Debug refusals with `ACL LOG` and `ACL WHOAMI`.
  https://redis.io/docs/latest/develop/reference/errors/common/noperm/

## TLS

- TLS covers client connections, replication (`tls-replication yes`) and the cluster bus (`tls-cluster yes`). `tls-port` adds to `port`: set `port 0` to close plaintext.

```
port 0
tls-port 6379
tls-cert-file /path/redis.crt
tls-key-file /path/redis.key
tls-ca-cert-file /path/ca.crt
```
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/encryption/
- Client certificates are required by default (mutual TLS) unless `tls-auth-clients no`. 8.6 can map a certificate to an ACL user (`tls-auth-clients-user`).
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/encryption/ and https://redis.io/docs/latest/develop/whats-new/8-6/
- TLS lowers per-instance throughput. Since 8.0 I/O threads work with TLS.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/encryption/

## Security fixes by release

- 8.10: ACL bypasses in `SORT`, `GEORADIUS`, `GEORADIUSBYMEMBER`, `XREAD`, `XREADGROUP`, CRLF injection in error replies, key-name leak in BCAST tracking invalidations.
  https://redis.io/docs/latest/develop/whats-new/8-10/
- 8.8: several remote code execution CVEs (unblock flow, `RESTORE`, Lua use-after-free) and `RESTORE` fixes for time series and probabilistic types.
  https://redis.io/docs/latest/develop/whats-new/8-8/
- 8.6: key-pattern ACL bypass in `MSETEX`, PII removed from ACL and server logs.
  https://redis.io/docs/latest/develop/whats-new/8-6/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `requirepass` shared password | ACL users (6.0) | https://redis.io/docs/latest/operate/oss_and_stack/management/security/ |
| `rename-command CONFIG ""` | ACL rules such as `-config`, `-@dangerous` | https://redis.io/docs/latest/operate/oss_and_stack/management/security/ |
| `-select +select\|0` first-argument rules | subcommand rules (`+config\|get`, 7.0); first-arg rules are deprecated since 7.0 | https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/ |
| `acl-pubsub-default allchannels` | `resetchannels` (7.0) | https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/ |
