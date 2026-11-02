# Resilience (@nestjs/resilience)

## Contents
- Model and registration
- Timeouts and the signal
- Retries
- Circuit breaker, fallback, bulkhead
- Policies inside services
- Other transports
- Errors and production

## Model and registration

- Decorators (`@Timeout`, `@Retry`, `@CircuitBreaker`, `@Bulkhead`, `@Fallback`) apply to entrypoints only: controllers, resolvers, message handlers, gateways. On provider methods they do nothing (bootstrap warns). Inside services use policy objects. https://docs.nestjs.com/reliability/resilience
- Register once in the root module. Presets name one dependency, and a preset's breaker, bulkhead and rate limit are shared by everything using it: https://docs.nestjs.com/reliability/resilience#register-the-module
  ```typescript
  ResilienceModule.forRoot({
    defaults: { timeout: '5s', retry: { attempts: 3, backoff: { delay: '200ms', maxDelay: '2s' } } },
    presets: {
      carrier: { timeout: '2s', retry: { attempts: 2 },
        circuitBreaker: { failureRateThreshold: 50, minimumCalls: 10, openDuration: '30s' } },
    },
  }),
  ```
- `defaults` only fill fields of stages a decorator or preset turns on. Defaults never turn a stage on. https://docs.nestjs.com/reliability/resilience#register-the-module
- Bad config (unknown preset, `NaN` duration from an unset env var, missing fallback method) fails at boot. https://docs.nestjs.com/reliability/resilience#register-the-module
- Register `IdempotencyModule` before `ResilienceModule`: the first global interceptor runs outermost, so replays are answered before retries and all retries share one key. https://docs.nestjs.com/reliability/resilience#retry-a-post-safely

## Timeouts and the signal

```typescript
@Get('quotes')
@Resilience('carrier')
@Timeout('1.5s')
getQuotes(@Query('orderId') orderId: string, @Signal() signal: AbortSignal) {
  return this.shippingService.getQuotes(orderId, signal);
}
```
https://docs.nestjs.com/reliability/resilience#put-a-time-budget-on-the-quotes-route

- Cancellation is cooperative: code that ignores the signal keeps running and its result is dropped. Pass the signal to every I/O call, or inject `ResilienceContext` where a parameter is impossible. https://docs.nestjs.com/reliability/resilience#put-a-time-budget-on-the-quotes-route
- The timeout is per attempt: total time is attempts times timeout plus backoff. It answers 504, not 408 (clients and proxies replay 408s). https://docs.nestjs.com/reliability/resilience#put-a-time-budget-on-the-quotes-route
- On retry only pipes, inner interceptors and the handler run again. Guards and middleware run once. Client errors (4xx) are never retried or counted. https://docs.nestjs.com/reliability/resilience#put-a-time-budget-on-the-quotes-route

## Retries

- HTTP retries apply only to GET, HEAD, OPTIONS (GraphQL: queries). A preset retry and even `@Retry()` on a POST are skipped. Opt in with `@Retry({ idempotent: true })` only when running the handler twice in one request is harmless. A preset cannot set `idempotent`. https://docs.nestjs.com/reliability/resilience#retry-a-post-safely
- `idempotent: true` (server re-runs within one request) and `@Idempotent()` (dedupes the client's retries) solve different problems: use both. https://docs.nestjs.com/reliability/resilience#retry-a-post-safely
- A 504 releases the idempotency key while the timed-out run may still reach the dependency: the downstream effect must be safe to repeat. https://docs.nestjs.com/reliability/resilience#retry-a-post-safely
- Retry at one layer only and turn off retries in SDKs and HTTP clients underneath. https://docs.nestjs.com/reliability/resilience#production-checklist

## Circuit breaker, fallback, bulkhead

- A breaker opens after at least `minimumCalls` (10) with `failureRateThreshold` (50 %) failed in the last 20 calls. Open calls answer 503 with `Retry-After`. After `openDuration` one probe passes. https://docs.nestjs.com/reliability/resilience#fail-fast-with-a-circuit-breaker
- A 408 or 429 from the dependency counts as a failure, while an `HttpException` with 408 or 429 thrown by the handler is a client error. Tune with `recordIf`. https://docs.nestjs.com/reliability/resilience#fail-fast-with-a-circuit-breaker
- Give every breaker a timeout: a hanging probe keeps a half-open breaker rejecting. https://docs.nestjs.com/reliability/resilience#production-checklist
- `@Fallback('flatRateQuotes', { handleIf: (e) => e instanceof CircuitOpenError })` is the outermost stage. Keep fallbacks cheap and never call the dependency they replace. https://docs.nestjs.com/reliability/resilience#fail-fast-with-a-circuit-breaker
- `@Bulkhead({ maxConcurrent: 2, maxQueue: 3, queueTimeout: '10s' })` answers 503 `BULKHEAD_FULL` when full. Defaults: 10 concurrent, no queue. https://docs.nestjs.com/reliability/resilience#cap-concurrent-csv-exports-with-a-bulkhead

## Policies inside services

```typescript
this.resiliencePolicy = resilience.preset('carrier'); // once, in the constructor
await this.resiliencePolicy.execute(
  ({ signal }) => this.carrierClient.getQuotes(order, signal),
  { source: 'RepricingService.repricePendingOrders' },
);
```
https://docs.nestjs.com/reliability/resilience#use-the-same-policies-inside-services

- A policy shares the preset's breaker with routes. Its retry always applies (no HTTP method to inspect): wrap only repeat-safe calls. https://docs.nestjs.com/reliability/resilience#use-the-same-policies-inside-services
- Undecorated entrypoints still map resilience errors (`CircuitOpenError` from a service gives 503). `mapErrors: false` disables it. https://docs.nestjs.com/reliability/resilience#use-the-same-policies-inside-services

## Other transports

- Hybrid apps need `inheritAppConfig: true` on `connectMicroservice()` or the global interceptor never reaches message handlers. https://docs.nestjs.com/reliability/resilience#other-transports
- On message and WebSocket handlers only a handler-level `@Retry()` enables retries. https://docs.nestjs.com/reliability/resilience#other-transports
- GraphQL failures fail only their field (HTTP 200), with `code`, `httpStatus`, `retryAfter` in `extensions`. https://docs.nestjs.com/reliability/resilience#other-transports

## Errors and production

- `ResilienceTimeoutError` 504, `CircuitOpenError` 503 + `Retry-After`, `BulkheadFullError` 503, `OutboundRateLimitError` 503 + `Retry-After`. https://docs.nestjs.com/reliability/resilience#errors
- State is per process: each instance has its own breakers and limits, reset on restart. https://docs.nestjs.com/reliability/resilience#production-checklist
- Events: `ResilienceEvents.events$` and `nestjs:resilience:<type>` diagnostics channels. https://docs.nestjs.com/reliability/resilience#observe-breaker-transitions
- Tests: `vi.useFakeTimers({ toFake: ['setTimeout', 'clearTimeout', 'Date'] })`, `vi.advanceTimersByTimeAsync()`, `circuitBreaker(name).trip()`. https://docs.nestjs.com/reliability/resilience#testing
