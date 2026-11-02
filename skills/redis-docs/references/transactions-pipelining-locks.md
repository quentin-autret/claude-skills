# Pipelining, transactions, compare-and-set and locks

Round-trip reduction, MULTI/EXEC and WATCH semantics, the 8.4 compare-and-set options, distributed locks, rate limiters and WAIT in Redis Open Source 8.10.

## Contents

- Cutting round trips
- MULTI/EXEC
- WATCH and optimistic locking
- Compare-and-set without WATCH (8.4+)
- Distributed locks
- Rate limiters
- WAIT
- Old patterns

## Cutting round trips

- Order of preference: variadic commands (`MSET`, `MGET`, `HSET` with many fields), then pipelining, then scripts when one reply feeds the next command.
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/
- A pipeline cannot use a reply to build a later write. Read-compute-write needs a script or a function.
  https://redis.io/docs/latest/develop/using-commands/pipelining/
- Cap pipelines at about 10k commands and read the replies before sending the next batch: the server holds the queued replies in memory.
  https://redis.io/docs/latest/develop/using-commands/pipelining/
- Bulk loading: generate raw RESP commands into a file and pipe it to `redis-cli --pipe`, not a client loop or netcat.
  https://redis.io/docs/latest/develop/clients/patterns/bulk-loading/
- Variadic commands with zero keys fail with "wrong number of arguments": check for an empty list first.
  https://redis.io/docs/latest/develop/reference/errors/common/err-wrong-number-of-arguments/

## MULTI/EXEC

- There is no rollback. A command that fails at EXEC time (for example WRONGTYPE) returns its error in the EXEC reply and the other commands still run.
  https://redis.io/docs/latest/develop/using-commands/transactions/
- An error while queueing (unknown command, wrong arity, OOM) makes EXEC discard everything with `EXECABORT`.

```
MULTI
FOO            -> (error) ERR unknown command 'FOO'
SET a 1        -> QUEUED
EXEC           -> (error) EXECABORT Transaction discarded because of previous errors.
```
  https://redis.io/docs/latest/develop/reference/errors/common/execabort/
- A client that disconnects before EXEC runs nothing. A crash mid-EXEC with AOF can leave a partial transaction that blocks restart until `redis-check-aof` repairs it.
  https://redis.io/docs/latest/develop/using-commands/transactions/
- A script can do everything a transaction does and is usually simpler and faster.
  https://redis.io/docs/latest/develop/using-commands/transactions/
- In a cluster all keys of a transaction must share one hash slot.
  https://redis.io/docs/latest/develop/reference/errors/common/crossslot/

## WATCH and optimistic locking

- Any change to a watched key aborts EXEC (null reply), including expiration and eviction. Commands queued in the same transaction do not. All keys are unwatched after EXEC and on disconnect.
  https://redis.io/docs/latest/develop/using-commands/transactions/
- Retry the whole read-WATCH-MULTI-EXEC sequence on a null reply.

```
WATCH mykey
val = GET mykey
MULTI
SET mykey $val+1
EXEC
```
  https://redis.io/docs/latest/develop/using-commands/transactions/
- In node-redis, call the read-only commands on the watched keys on the usual `client` object and the transaction commands on the object created with `multi()`.
  https://redis.io/docs/latest/develop/clients/nodejs/transpipe/

## Compare-and-set without WATCH (8.4+)

- For compare-and-set or compare-and-delete on a string, `SET ... IFEQ` and `DELEX ... IFEQ` replace WATCH loops and Lua, and the docs call them simpler and faster. `IFDEQ` compares an XXH3 digest obtained with `DIGEST`.

```
SET bike:1 Ares IFEQ Deimos
DELEX resource_name IFEQ my_random_value
```
  https://redis.io/docs/latest/develop/using-commands/transactions/ and https://redis.io/docs/latest/develop/data-types/strings/

## Distributed locks

- Acquire with a random token and a TTL: `SET resource token NX PX 30000`. Release only if the token matches: `DELEX resource IFEQ token` on 8.4+. A plain `DEL` can delete a lock another client took after yours expired.
  https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/
- Before 8.4, release with the compare-and-delete script:

```
if redis.call("get",KEYS[1]) == ARGV[1] then return redis.call("del",KEYS[1]) else return 0 end
```
  https://redis.io/docs/latest/commands/set/
- The docs discourage the single-instance `SET NX` lock in favor of Redlock. A lock on a primary with replica failover is unsafe: the primary can crash before the key replicates and a second client acquires it.
  https://redis.io/docs/latest/commands/set/ and https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/
- Redlock: 5 independent primaries without replication, a small per-node timeout (about 5-50 ms for a 10 s TTL), success only with a majority and elapsed time under the validity, random retry delay, unlock all nodes on failure.
  https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/
- Redlock relies on wall clocks: a clock jump can hand the lock to two clients. Use fencing tokens. A restarted node without persistence breaks exclusivity: use AOF with `fsync=always` or delay restarts beyond the max TTL.
  https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/

## Rate limiters

- Fixed window: `INCR key` + `EXPIRE key window`. On 8.8+, `INCREX` combines increment, bounds and expiration in one atomic command, which removes the need for a multi-command transaction.
  https://redis.io/docs/latest/develop/use-cases/rate-limiter/ and https://redis.io/docs/latest/develop/whats-new/8-8/
- Sliding window and token bucket: keep the read-decide-update in a Lua script so concurrent requests cannot double-spend.
  https://redis.io/docs/latest/develop/use-cases/rate-limiter/
- Scope keys to the limited dimension (user, IP, API key, tenant). Per-process counters behind a load balancer are bypassed.
  https://redis.io/docs/latest/develop/use-cases/rate-limiter/

## WAIT

- `WAIT numreplicas timeout` returns how many replicas acknowledged, on success and on timeout alike: compare it with the number you need. Timeout 0 blocks forever. Inside MULTI it does not block, and since 7.0 neither does it in any context that forbids blocking, such as scripts.
  https://redis.io/docs/latest/commands/wait/
- WAIT does not make Redis strongly consistent. An acknowledged write can still be lost on failover.
  https://redis.io/docs/latest/commands/wait/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| WATCH/MULTI/EXEC retry for a string compare-and-set | `SET ... IFEQ/IFDEQ`, `DELEX` (8.4) | https://redis.io/docs/latest/develop/using-commands/transactions/ |
| Lua compare-and-delete to release a lock | `DELEX key IFEQ token` (8.4) | https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/ |
| `INCR` + `EXPIRE` in MULTI for a window counter | `INCREX` (8.8) | https://redis.io/docs/latest/develop/whats-new/8-8/ |
| netcat to mass-import data | `redis-cli --pipe` | https://redis.io/docs/latest/develop/clients/patterns/bulk-loading/ |
