# Persistence, backups, configuration and host setup

RDB and AOF choice, fsync, switching to AOF live, backups (including the 8.10 BACKUP command), configuration files, signals and Linux host settings for self-managed Redis Open Source 8.10.

## Contents

- Choosing persistence
- AOF behavior
- Switching from RDB to AOF
- Backups
- RDB and AOF failure modes
- Configuration
- Shutdown and signals
- Host setup
- Old patterns

## Choosing persistence

- Default for data you care about: RDB + AOF together, for safety comparable to PostgreSQL. RDB alone if a few minutes of loss is acceptable. AOF alone is discouraged: RDB snapshots make backups and restarts faster.
  https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- With both enabled, restart rebuilds from the AOF, the most complete one.
  https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- RDB loses the last minutes on crash, power loss or `kill -9`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- A primary without persistence and with auto-restart wipes its replicas: it comes back empty and they resync from it. Enable persistence or disable auto-restart, even under Sentinel.
  https://redis.io/docs/latest/operate/oss_and_stack/management/replication/

## AOF behavior

- Keep `appendfsync everysec` (loses about 1 s at most). `always` is very slow but groups fsyncs across clients. `no` lets the OS flush (about every 30 s on Linux).
  https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- Since 7.0 the AOF is multi-part (base + incremental files + manifest in `appenddirname`).
  https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- A truncated AOF loads anyway by default (`aof-load-truncated`) and drops the last bad command. A corruption in the middle aborts startup: back up the file, run `redis-check-aof` WITHOUT `--fix` first, because `--fix` may discard everything from the bad point to the end of the file.
  https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- `BGREWRITEAOF` during an RDB save is scheduled. During a running rewrite it errors and is not scheduled. A failed rewrite loses nothing.
  https://redis.io/docs/latest/commands/bgrewriteaof/

## Switching from RDB to AOF

Never just edit the config and restart: the docs warn it loses data. Switch on the live server:

```
redis-cli config set appendonly yes
redis-cli config set save ""        # optional, drop RDB
redis-cli config rewrite
redis-cli info persistence
```
Wait for `aof_rewrite_in_progress:0`, `aof_rewrite_scheduled:0`, `aof_last_bgrewrite_status:ok` before any restart, then compare key counts.
https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/

## Backups

- RDB files are safe to copy while Redis runs: they are written under a temp name and renamed atomically.
  https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- Routine from the docs: hourly snapshots kept 48 hours, daily snapshots kept 1-2 months, at least one RDB a day shipped outside the data center (or at least the physical machine), possibly encrypted, size checked and possibly SHA1 checked, with an alert when backups stop arriving.
  https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- Copying the AOF directory during a rewrite can produce an invalid backup. Pause rewrites first:

```
CONFIG SET auto-aof-rewrite-percentage 0
INFO persistence            # wait for aof_rewrite_in_progress:0
# copy or hard-link the appenddirname files
CONFIG SET auto-aof-rewrite-percentage <previous>
```
  https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- Since 8.10, online backups with `BACKUP START`, `BACKUP LIST`, `BACKUP SEAL`, `BACKUP CLEANUP` (plus `BACKUP STATUS` and `BACKUP ABORT`). It reuses the multi-part AOF format, never stops writes, works with AOF on or off and can be staggered across cluster nodes. `backupdirname` (startup only) and `backup-sealed-ttl` (0 = no auto cleanup) configure it.
  https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- Restore with the startup-only `preload-file aof:<manifest>` or `preload-file rdb:<file>`. It loads only that file, aborts startup if loading fails, then continues in the configured persistence mode.
  https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/

## RDB and AOF failure modes

- A failed or killed RDB child makes Redis reply to writes with a `MISCONFIG` error (the page's spelling) until an RDB file can be created again. A killed AOF rewrite child is just retried. `SIGUSR1` kills the RDB child without the error state.
  https://redis.io/docs/latest/operate/oss_and_stack/reference/signals/
- `BGSAVE` errors while another save or rewrite runs. `BGSAVE SCHEDULE` returns OK when an AOF rewrite is running and schedules the save for the next opportunity instead.
  https://redis.io/docs/latest/commands/bgsave/
- `fork()` can stall clients for some milliseconds, up to about 1 s on a very big dataset with a modest CPU.
  https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- `FLUSHALL` also empties the RDB file.
  https://redis.io/docs/latest/commands/flushall/

## Configuration

- Production runs from a `redis.conf`. The built-in defaults are for development.
  https://redis.io/docs/latest/operate/oss_and_stack/management/config/
- `CONFIG SET` never touches the file, so the old value returns at restart: follow it with `CONFIG REWRITE`. `CONFIG SET appendonly yes` starts a background rewrite.
  https://redis.io/docs/latest/operate/oss_and_stack/management/config/ and https://redis.io/docs/latest/commands/config-set/
- 8.10 ships a single `redis.conf`. 8.0 to 8.8 shipped `redis.conf` plus `redis-full.conf`, which loaded the Search, time series and probabilistic modules.
  https://redis.io/docs/latest/operate/oss_and_stack/management/config/
- Module settings (`search-*`, `ts-*`, `bf-*`, `cf-*`) can be set directly in `redis.conf` since 8.0, and by `CONFIG SET` for the run-time ones (most of them), instead of as `loadmodule` arguments.
  https://redis.io/docs/latest/develop/ai/search-and-query/administration/configuration/ and https://redis.io/docs/latest/develop/data-types/timeseries/configuration/
- Docker: mount a data volume (`-v /local-data/:/data`) and a writable config dir.
  https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/docker/

## Shutdown and signals

- `SHUTDOWN`, SIGTERM and SIGINT wait up to `shutdown-timeout` (10 s) for lagging replicas, fsync the AOF and save the RDB if configured. If that save fails, the shutdown aborts and the server keeps running.
  https://redis.io/docs/latest/operate/oss_and_stack/reference/signals/
- A running script delays shutdown. In a diskless setup, fail over before stopping a primary.
  https://redis.io/docs/latest/operate/oss_and_stack/reference/signals/

## Host setup

- `vm.overcommit_memory = 1`, transparent huge pages disabled, swap enabled and as large as RAM so overruns show as latency instead of an OOM kill.
  https://redis.io/docs/latest/operate/oss_and_stack/management/admin/
- EBS-like network volumes can have high latency: be careful with persistence on them.
  https://redis.io/docs/latest/operate/oss_and_stack/management/admin/
- Unexplained crashes: test RAM with `redis-server --test-memory`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/troubleshooting/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Single-file AOF, manual copies | Multi-part AOF (7.0) | https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/ |
| Pausing rewrites to copy AOF files | `BACKUP START` / `SEAL` (8.10) | https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/ |
| `redis-full.conf` with `loadmodule` | single `redis.conf` (8.10) | https://redis.io/docs/latest/operate/oss_and_stack/management/config/ |
| Module options after `loadmodule` | `CONFIG SET` and `redis.conf` names (8.0) | https://redis.io/docs/latest/develop/data-types/probabilistic/configuration/ |
