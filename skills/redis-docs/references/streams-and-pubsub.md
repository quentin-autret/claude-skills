# Streams, consumer groups, Pub/Sub and keyspace notifications

Delivery guarantees, consumer group loops, trimming, idempotent producers, sharded Pub/Sub and keyspace notifications in Redis Open Source 8.10.

## Contents

- Choosing the messaging tool
- Writing and trimming streams
- Consumer groups
- Recovering and releasing messages
- Idempotent producers (8.6+)
- Pub/Sub
- Keyspace notifications
- Old patterns

## Choosing the messaging tool

- Pub/Sub is at-most-once: a subscriber that is offline or errors loses the message for good. Use Streams for persistence, replay or at-least-once delivery.
  https://redis.io/docs/latest/develop/pubsub/
- Consumer group when messages are split across workers. Plain `XREAD` when every client must see every message.
  https://redis.io/docs/latest/commands/xreadgroup/
- Idle-time checks on claims avoid trivial re-processing, but in the general case you cannot obtain exactly-once processing.
  https://redis.io/docs/latest/develop/data-types/streams/
- Replication is asynchronous: entries and group state can be missing after failover. `WAIT` reduces but does not remove the risk. For durability use AOF with a strong fsync policy.
  https://redis.io/docs/latest/develop/data-types/streams/
- One stream lives on one node. To scale, shard across several keys. With one stream and N consumers, events about the same item can be processed out of order.
  https://redis.io/docs/latest/develop/data-types/streams/

## Writing and trimming streams

- Let Redis generate IDs with `*`. An explicit ID must exceed the current top ID. IDs stay monotonic across clock jumps and failovers.
  https://redis.io/docs/latest/commands/xadd/
- Cap on write with approximate trimming: `~` trims whole macro nodes and is much cheaper than exact `=` (the default), leaving a few extra entries.

```
XADD mystream MAXLEN ~ 1000 * field value
XTRIM mystream MINID ~ 649085820
```
  https://redis.io/docs/latest/commands/xadd/ and https://redis.io/docs/latest/commands/xtrim/
- `MAXLEN` caps by entry count. To cap by age, trim by `MINID`: auto-generated IDs start with the Unix time in milliseconds.
  https://redis.io/docs/latest/commands/xtrim/ and https://redis.io/docs/latest/commands/xadd/
- XADD is O(N) when it evicts N entries. `XTRIM ... LIMIT 0` removes the work cap entirely.
  https://redis.io/docs/latest/commands/xadd/ and https://redis.io/docs/latest/commands/xtrim/
- Default trimming is `KEEPREF`: entries still pending in a group can be trimmed, and reads of them return nil payloads. With several groups, trim with `ACKED` (8.2) so entries go only once every group acknowledged them.
  https://redis.io/docs/latest/commands/xreadgroup/ and https://redis.io/docs/latest/develop/data-types/streams/
- A stream is never auto-deleted at length zero, unlike other types.
  https://redis.io/docs/latest/develop/data-types/streams/
- `$` is the last ID of this stream, `+` the greatest possible ID: they are not interchangeable. Paginate with `XRANGE key (lastID + COUNT n`.
  https://redis.io/docs/latest/develop/data-types/streams/

## Consumer groups

- Create idempotently at startup and catch `BUSYGROUP`. Without `MKSTREAM`, the stream must already exist. `$` reads only new entries, `0` the whole history.

```
XGROUP CREATE mystream mygroup $ MKSTREAM
XREADGROUP GROUP mygroup consumer-1 COUNT 10 BLOCK 2000 STREAMS mystream >
XACK mystream mygroup 1526569495631-0
```
  https://redis.io/docs/latest/commands/xgroup-create/ and https://redis.io/docs/latest/develop/reference/errors/common/busygroup/
- Consumer loop: on start read your own history with ID `0` until empty (replays unacked entries after a crash), then switch to `>`. Any ID other than `>` returns only this consumer's pending entries and ignores `BLOCK`, `NOACK` and `CLAIM`.
  https://redis.io/docs/latest/commands/xreadgroup/
- Always `XACK` after processing, or the entry stays in the pending list (PEL) and keeps memory. `NOACK` skips the PEL, so a crash loses the message.
  https://redis.io/docs/latest/commands/xack/ and https://redis.io/docs/latest/commands/xreadgroup/
- `STREAMS` is the last option: keys, then IDs. Consumer names are case sensitive, created on first use, and must be unique.
  https://redis.io/docs/latest/develop/data-types/streams/
- `XREADGROUP` is flagged as a write command. `NOGROUP` also appears when the stream key was deleted or expired, taking its groups with it. `XGROUP DESTROY` drops the PEL too.
  https://redis.io/docs/latest/commands/xreadgroup/ and https://redis.io/docs/latest/develop/reference/errors/common/nogroup/ and https://redis.io/docs/latest/develop/reference/errors/common/busygroup/
- 8.10: `MAXCOUNT` and `MAXSIZE` cap the total entries or bytes of a multi-stream read.
  https://redis.io/docs/latest/commands/xreadgroup/

## Recovering and releasing messages

- A dead consumer keeps its pending entries forever unless someone claims them. Default: `XAUTOCLAIM`, looping on the returned cursor and restarting from `0-0` once done.

```
XAUTOCLAIM mystream mygroup Alice 3600000 0-0 COUNT 25
```
  https://redis.io/docs/latest/commands/xautoclaim/
- XAUTOCLAIM scans at most COUNT x 10 entries, so it can claim fewer than COUNT. Claiming increments the delivery counter unless `JUSTID` is used. Entries trimmed away are listed separately: send them to a dead-letter store.
  https://redis.io/docs/latest/commands/xautoclaim/ and https://redis.io/docs/latest/develop/use-cases/streaming/
- 8.4: `XREADGROUP ... CLAIM min-idle-time ... >` claims idle entries and reads new ones in one call.
  https://redis.io/docs/latest/commands/xreadgroup/
- Poison messages: when the delivery counter (4th field of `XPENDING`) passes a threshold, move the entry to another stream and alert.
  https://redis.io/docs/latest/develop/data-types/streams/ and https://redis.io/docs/latest/commands/xpending/
- 8.8: `XNACK` releases pending messages at shutdown or on failure instead of waiting for idle timeouts (`SILENT`, `FAIL`, `FATAL` modes).

```
XNACK mystream mygroup FAIL IDS 1 1526569498055-0
```
  https://redis.io/docs/latest/develop/data-types/streams/
- `XPENDING key group IDLE ms - + count [consumer]` filters to entries idle for at least `ms`. `XINFO` replies gain fields over versions (IDMP fields in 8.6, `nacked-count` in 8.8): read them by name.
  https://redis.io/docs/latest/commands/xpending/ and https://redis.io/docs/latest/commands/xinfo-stream/

## Idempotent producers (8.6+)

- Producers that retry after a disconnect add `IDMP pid iid` (you supply the iid, faster) or `IDMPAUTO pid` (Redis hashes the fields). A replayed pair returns the original ID. Only with the `*` ID.

```
XCFGSET mystream IDMP-DURATION 300 IDMP-MAXSIZE 1000
XADD mystream IDMP producer1 msg1 * field value
```
  https://redis.io/docs/latest/develop/data-types/streams/idempotency/ and https://redis.io/docs/latest/commands/xadd/
- Keep producer IDs short, unique and stable across restarts. `IDMP-DURATION` must cover the producer's recovery time, and `IDMP-MAXSIZE` wins over it (defaults 100 s and 100 iids per producer). Changing these values clears the key's dedup map. Check `iids-duplicates` in `XINFO STREAM`.
  https://redis.io/docs/latest/develop/data-types/streams/idempotency/

## Pub/Sub

- Pub/Sub ignores database numbers: prefix channels with the environment if needed.
  https://redis.io/docs/latest/develop/pubsub/
- On RESP2, a subscribed connection accepts only subscribe commands, PING, QUIT and RESET: give subscriptions their own connection. RESP3 connections can run any command while subscribed.
  https://redis.io/docs/latest/develop/pubsub/
- A client subscribed to `foo` and `f*` receives the message twice.
  https://redis.io/docs/latest/develop/pubsub/
- A slow subscriber is disconnected past its output buffer limit (hard 32 MB, soft 8 MB over 60 s).
  https://redis.io/docs/latest/develop/reference/clients/
- In Cluster, use sharded Pub/Sub (`SSUBSCRIBE`/`SPUBLISH`, 7.0): messages stay in the shard instead of reaching every node. Channels of one `SSUBSCRIBE` call must share a slot.
  https://redis.io/docs/latest/develop/pubsub/ and https://redis.io/docs/latest/commands/ssubscribe/

## Keyspace notifications

- Off by default (CPU cost). `notify-keyspace-events` needs `K` or `E` or nothing is sent. `A` excludes `m`, `n`, `o`, `c` (miss, new, overwritten, type-changed).

```
CONFIG SET notify-keyspace-events KEA
PSUBSCRIBE '__key*__:*'
```
  https://redis.io/docs/latest/develop/pubsub/keyspace-notifications/
- Fire and forget: events during a disconnect are lost. `expired` fires when Redis actually deletes the key, which can be late. Events fire only on real changes.
  https://redis.io/docs/latest/develop/pubsub/keyspace-notifications/
- In a cluster, events are per node: subscribe on every node.
  https://redis.io/docs/latest/develop/pubsub/keyspace-notifications/
- 8.8 subkey notifications (flags `S`, `T`, `I`, `V`, hash commands only) are enabled separately from keyspace notifications.
  https://redis.io/docs/latest/develop/pubsub/subkeyspace-notifications/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `XPENDING` + `XCLAIM` recovery loop | `XAUTOCLAIM` (6.2), or `XREADGROUP ... CLAIM` (8.4) | https://redis.io/docs/latest/commands/xreadgroup/ |
| App logic deleting after all groups ack | `ACKED`/`DELREF` on `XADD`, `XTRIM`, `XDELEX`, `XACKDEL` (8.2) | https://redis.io/docs/latest/develop/data-types/streams/ |
| Waiting for idle timeouts at shutdown | `XNACK` (8.8) | https://redis.io/docs/latest/develop/data-types/streams/ |
| Producer-side dedup of retried XADD | `IDMP`/`IDMPAUTO` (8.6) | https://redis.io/docs/latest/develop/data-types/streams/idempotency/ |
| Global `PUBLISH` at volume in Cluster | `SPUBLISH`/`SSUBSCRIBE` (7.0) | https://redis.io/docs/latest/develop/pubsub/ |
