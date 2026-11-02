# Webhooks (@nestjs/webhooks)

## Contents
- Model
- Registration and storage
- Sending
- Delivery behavior
- Receiving and verifying
- Testing

## Model

- Sending with a plain `fetch()` after commit is lost if the process dies and sent even if the transaction rolled back. `webhooks.dispatch(tx, ...)` adds the message to the outbox in your transaction. https://docs.nestjs.com/http/webhooks https://docs.nestjs.com/http/webhooks#send-ordershipped-and-ordercancelled-after-the-commit
  ```typescript
  await this.webhooks.dispatch(tx, { type: 'order.cancelled', tenant: row.partnerId, data });
  this.webhooks.notify(); // deliver now instead of at the next poll
  ```
- `WebhooksModule` requires `OutboxModule` (startup fails without it). Incoming webhooks are deduplicated by the outbox inbox. Both modules are global. https://docs.nestjs.com/http/webhooks#register-the-modules
- Pass the partner id as `tenant` on every call. Another tenant's endpoint behaves as missing (404). https://docs.nestjs.com/http/webhooks#let-partners-subscribe
- Declare `eventTypes`: `dispatch()` and new endpoints refuse other types. https://docs.nestjs.com/http/webhooks#register-the-modules
- A receive-only service: `WebhooksModule.forRootAsync({ outgoing: false, ... })` and `OutboxModule.forRoot({ relay: { enabled: false } })`. https://docs.nestjs.com/http/webhooks#module-options https://docs.nestjs.com/http/webhooks#send-ordershipped-and-ordercancelled-after-the-commit

## Registration and storage

- Default storage is in memory. With `NODE_ENV=production` and no store, startup fails unless `allowInMemoryStorage: true`. https://docs.nestjs.com/http/webhooks#keep-webhooks-in-your-database
- Register `PostgresWebhookStore` (or `MySqlWebhookStore`) once in the root module with a factory: https://docs.nestjs.com/http/webhooks#keep-webhooks-in-your-database
  ```typescript
  { provide: PostgresWebhookStore, inject: [getDrizzleToken(), WebhooksStorage],
    useFactory: (db: Database, s: WebhooksStorage) =>
      new PostgresWebhookStore({ executor: fromDrizzle(db) }, s) }
  ```
- Executors come from the store's own subpath (`@nestjs/webhooks/postgres` or `/mysql`). The other dialect's executor does not compile. https://docs.nestjs.com/http/webhooks#postgresql-and-mysql-stores
- The store creates its own schema (`nest_webhooks`). No entities or Prisma models. `migrate` defaults to `true` except with `NODE_ENV=production`: run `npx @nestjs/webhooks migrate` before deploy and `status` (exits 1 when behind) in CI. https://docs.nestjs.com/http/webhooks#keep-webhooks-in-your-database
- Pass your transaction object, never the bare database: Drizzle `tx`, TypeORM `EntityManager` from `dataSource.transaction()`, a pg client after `BEGIN`, a Prisma interactive transaction client. https://docs.nestjs.com/http/webhooks#keep-webhooks-in-your-database https://docs.nestjs.com/http/webhooks#postgresql-and-mysql-stores
- PostgreSQL stores rely on READ COMMITTED and fail at startup if `default_transaction_isolation` is stricter. MySQL 8.4 or 9.x only, MariaDB is refused. https://docs.nestjs.com/http/webhooks#keep-webhooks-in-your-database
- Call `app.enableShutdownHooks()` so the relay and worker finish in-flight work. https://docs.nestjs.com/http/webhooks#production-checklist

## Sending

- `WebhookEndpoints.create()` returns a `whsec_...` secret once. `list()`/`get()` never include it. https://docs.nestjs.com/http/webhooks#let-partners-subscribe
- Partner URLs are untrusted: non-https URLs, credentials, private and metadata addresses are refused (400 `InvalidWebhookEndpointError`), checked again at every send, redirects never followed. Keep `delivery.allowHttp` and `allowPrivateNetworks` off in production. https://docs.nestjs.com/http/webhooks#let-partners-subscribe https://docs.nestjs.com/http/webhooks#transports
- Signatures follow Standard Webhooks: `webhook-id`, `webhook-timestamp`, `webhook-signature: v1,<base64 HMAC>`. https://docs.nestjs.com/http/webhooks#send-ordershipped-and-ordercancelled-after-the-commit
- `rotateSecret()` keeps the old secret signing for 24 h (`secretRotationOverlap`). https://docs.nestjs.com/http/webhooks#rotate-secrets
- Version payloads by adding fields only. For a breaking change add a new type (`order.shipped.v2`) and dispatch both in one transaction. https://docs.nestjs.com/http/webhooks#production-checklist
- Map package errors with an `@Catch(WebhooksError)` filter extending `BaseExceptionFilter`, registered with `APP_FILTER`. https://docs.nestjs.com/http/webhooks#let-partners-subscribe

## Delivery behavior

- Defaults: 10 attempts with backoff from `'5s'` by factor 4 up to `'1d'` (about two days), `delivery.timeout` `'15s'`, endpoint disabled after `'5d'` of failures. https://docs.nestjs.com/http/webhooks#module-options
- Only 2xx succeeds. 3xx is a failure. 410 disables the endpoint. 429/502/503/504 pause the endpoint. https://docs.nestjs.com/http/webhooks#retries-the-delivery-log-and-replay
- At least once, unordered: receivers deduplicate by `webhook-id` and use the body `timestamp` to drop stale events. https://docs.nestjs.com/http/webhooks#delivery-guarantees
- Run the worker in a few instances (`worker.enabled: false` elsewhere). Nothing prunes the log for you: `deliveries.prune('30d')` from a cron job. Alert on `stats().lagMs`. https://docs.nestjs.com/http/webhooks#production-checklist

## Receiving and verifying

- Create the app with `rawBody: true`. Without it, JSON requests get a 500. https://docs.nestjs.com/http/webhooks#receive-the-payment-providers-webhooks https://docs.nestjs.com/http/webhooks#errors
  ```typescript
  receivers: {
    payments: { scheme: 'standard', secret: process.env.PAYMENTS_WEBHOOK_SECRET! },
    carrier: { scheme: 'stripe', header: 'Carrier-Signature', secret: process.env.CARRIER_WEBHOOK_SECRET! },
  },
  // route
  @Post() @VerifyWebhook('payments')
  receive(@IncomingWebhook() webhook: IncomingWebhook<Event>) {}
  ```
- Verification checks headers, constant-time signature over the raw body, a 5 minute timestamp window, and JSON. Failures answer a generic 401 (a 400 for a signed body that isn't JSON), details only in the `verification-failed` event. https://docs.nestjs.com/http/webhooks#receive-the-payment-providers-webhooks
- Bodies over the parser limit (100 kB Express, 1 MiB Fastify) get 413 before verification. Raise with `useBodyParser()`. https://docs.nestjs.com/http/webhooks#receive-the-payment-providers-webhooks
- The inbox answers a known id with an empty 200 without running the handler. The id is recorded after success, so a throwing handler reruns on retry. For money, use exactly-once in your transaction: https://docs.nestjs.com/http/webhooks#receive-the-payment-providers-webhooks
  ```typescript
  const result = await this.db.transaction(async (tx) =>
    webhook.processInTransaction(tx, () => this.ordersService.markPaid(tx, id, paymentId, amount)),
  );
  ```
- GitHub signs neither a timestamp nor an id, and `X-GitHub-Delivery` is unsigned: key dedup on the body with the receiver `id` option. https://docs.nestjs.com/http/webhooks#other-senders
- Custom schemes compare with `WebhookSignatureScheme.matches()`, never `===`. https://docs.nestjs.com/http/webhooks#other-senders
- Keep the receiver `consumer` name stable (https://docs.nestjs.com/http/webhooks#module-options). Prune the inbox (`OutboxInbox.prune()`) on a window longer than any sender's retry schedule. https://docs.nestjs.com/http/webhooks#production-checklist

## Testing

- PGlite plus `OUTBOX_RELAY=off` and `WEBHOOKS_WORKER=off`, override `WebhookTransport` with `InMemoryWebhookTransport`, drive with `relay.runOnce()` then `worker.runOnce()`. Sign test requests with `signWebhook()` from `@nestjs/webhooks/testing`. https://docs.nestjs.com/http/webhooks#testing
