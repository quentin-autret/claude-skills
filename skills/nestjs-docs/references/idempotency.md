# Idempotency keys (@nestjs/idempotency)

## Contents
- Registration
- Behavior
- What is stored
- Store
- Encryption
- GraphQL and microservices
- Production

## Registration

- The package implements the IETF `Idempotency-Key` draft for HTTP, GraphQL mutations and message handlers with one decorator. A status check alone cannot stop double charges (two concurrent requests both see `pending`). https://docs.nestjs.com/reliability/idempotency
  ```typescript
  IdempotencyModule.forRoot({
    scope: (req: { user?: User }) => req.user?.id, // runs after guards
  }),
  // ...
  @Post(':id/pay')
  @Idempotent({ required: true })
  ```
  https://docs.nestjs.com/reliability/idempotency#make-the-payment-endpoint-idempotent
- Without `scope`, all clients share one namespace and one user's key could replay another user's receipt. `scope: false` only where a shared namespace is deliberate (webhooks keyed by provider ids). https://docs.nestjs.com/reliability/idempotency#make-the-payment-endpoint-idempotent
- `scope` receives the request for HTTP and GraphQL but the payload for microservices. In mixed apps pass `{ http, graphql, rpc }`. https://docs.nestjs.com/reliability/idempotency#tune-ttls-and-failure-handling

## Behavior

- `required: true` rejects a missing key with 400 `IDEMPOTENCY_KEY_REQUIRED`. https://docs.nestjs.com/reliability/idempotency#make-the-payment-endpoint-idempotent
- Same key still running: 409 `IDEMPOTENCY_KEY_IN_USE` with `Retry-After`. Completed: stored response plus `Idempotent-Replayed: true`, handler not run. Same key with another body or URL: 422 `IDEMPOTENCY_KEY_REUSED`. https://docs.nestjs.com/reliability/idempotency#make-the-payment-endpoint-idempotent
- Clients create the key when the user decides (tap on Pay) and persist it with the pending action. Check `code`, not only the status: a 409 can also be a final "already paid". https://docs.nestjs.com/reliability/idempotency#send-keys-from-the-mobile-app
- CORS: add `Idempotency-Key` to `allowedHeaders`, `Idempotent-Replayed` and `Retry-After` to `exposedHeaders`. https://docs.nestjs.com/reliability/idempotency#production-checklist

## What is stored

- Status below 500 is stored (`storeIf` default). 5xx and unknown errors release the key. `StreamableFile`, streams and `@Res()` without `passthrough` are not stored. https://docs.nestjs.com/reliability/idempotency#tune-ttls-and-failure-handling
- A 5xx after the side effect (charge succeeded, `markPaid()` failed) releases the key, and a retry charges again. Pass the payment provider a key derived from user id and idempotency key, or widen `storeIf`, or use the outbox. https://docs.nestjs.com/reliability/idempotency#tune-ttls-and-failure-handling
- A replayed error is a plain `HttpException`: filters checking `instanceof` a custom class see the base class. https://docs.nestjs.com/reliability/idempotency#tune-ttls-and-failure-handling
- The interceptor must stay outside other global interceptors. `APP_INTERCEPTOR` providers of `AppModule` register before those of imported modules and run outside it. It refuses to start when one of them is `ClassSerializerInterceptor` (a replay would expose `@Exclude()` fields): register those with `useGlobalInterceptors()` or in a module imported after `IdempotencyModule`. https://docs.nestjs.com/reliability/idempotency#production-checklist
- Defaults: `ttl` `'24h'` (must exceed the client retry window), `lockTtl` `'60s'`, `retryAfter` `'1s'`. https://docs.nestjs.com/reliability/idempotency#tune-ttls-and-failure-handling

## Store

- The default `InMemoryIdempotencyStore` loses records on deploy and is not shared. With `NODE_ENV=production` the app refuses to start without a store. The package ships no production store. https://docs.nestjs.com/reliability/idempotency#store-keys-in-postgresql
- Implement `acquire`, `complete`, `release`, `extend` (durations in whole ms), each atomic with the owner check in the same statement. Register with `storage.registerSource(this)` from a singleton constructor, before the module initializes. https://docs.nestjs.com/reliability/idempotency#the-store-contract
- PostgreSQL: primary key is the SHA-256 of the record key, `response` is `json` not `jsonb` (jsonb rejects `\u0000`), `expires_at` is a `bigint` of epoch ms. `acquire()` is one `INSERT ... ON CONFLICT DO UPDATE ... WHERE expires_at <= now RETURNING`. Call `prune()` on a schedule. https://docs.nestjs.com/reliability/idempotency#store-keys-in-postgresql
- Redis: one Lua script per method, `maxmemory-policy noeviction`, a key prefix per service. https://docs.nestjs.com/reliability/idempotency#store-keys-in-redis-instead
- Test with `idempotencyStoreContract(() => store, { advanceTime, concurrent: true })` on a pooled server. https://docs.nestjs.com/reliability/idempotency#the-store-contract

## Encryption

- `encryption: { keys: process.env.IDEMPOTENCY_KEYS!.split(',') }` in `forRootAsync()`. Keys are 32+ characters, newest first. Turn it on before the first record: switching later makes existing records unreadable until they expire (500 `IDEMPOTENCY_RECORD_UNREADABLE`). https://docs.nestjs.com/reliability/idempotency#encrypt-stored-receipts

## GraphQL and microservices

- GraphQL reads the key from an `idempotencyKey` argument, then the header. Replays do not set `Idempotent-Replayed`, and retries must send the same document (aliases are part of the key). https://docs.nestjs.com/reliability/idempotency#pay-through-graphql
- Events: derive the key from data (`order-paid:<orderId>`) with `@Idempotent({ required: true, keyFrom: { payload: 'eventId' } })`, `ttl: '7d'`, no `scope`. Do not acknowledge a delivery rejected as in use (RabbitMQ, Kafka). https://docs.nestjs.com/reliability/idempotency#deduplicate-orderpaid-events

## Production

- Register a shared store even for one instance. Use `required: true` on every endpoint that moves money. Set `scope` on every client-facing endpoint. Keep the business state check as a conditional update: two taps are two keys. Alert on `lock-lost` events. https://docs.nestjs.com/reliability/idempotency#production-checklist
