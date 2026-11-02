# HTTP client (@nestjs/http-client)

## Contents
- Registration
- Requests
- Timeouts and cancellation
- Retries
- Errors
- Interceptors, streaming, redirects, security
- Testing
- Old patterns: @nestjs/axios

## Registration

- `@nestjs/http-client` is built on Node `fetch` and replaces `HttpModule` from `@nestjs/axios` as the documented client. Every method returns a Promise, not an Observable. https://docs.nestjs.com/application/http-client
  ```typescript
  HttpClientModule.register({ baseUrl: 'https://api.example.com/v1', timeout: '5s' })
  const { data } = await this.http.get<Cat[]>('/cats');
  ```
- `baseUrl` and a relative path join with one `/` and keep the `/v1` prefix (unlike `new URL()`). Without `baseUrl`, URLs must be absolute. https://docs.nestjs.com/application/http-client#getting-started
- Each `register()` is local to its module unless `isGlobal: true`. Named clients: `register({ name: 'github', ... })` and `@InjectHttpClient('github')`. https://docs.nestjs.com/application/http-client#named-clients
- `HttpClientModule.forRoot()` sets app-wide defaults only (no client, no `baseUrl`). Client settings win, except headers (merged, `null` removes), retry (merged by field) and interceptors (`forRoot()` ones run first). https://docs.nestjs.com/application/http-client#application-wide-defaults
- In `registerAsync()`, `name`, `isGlobal`, `imports` and interceptor classes sit next to `useFactory`. A factory returning one of them stops startup. https://docs.nestjs.com/application/http-client#async-configuration

## Requests

- `params` fills `:name` segments, encoded. It throws for a missing value, an unknown key, or a value that would change the path (`''`, `.`, `..`). https://docs.nestjs.com/application/http-client#making-requests
- `query` repeats keys for arrays and skips `null`/`undefined`. `json` serializes and sets the content type. Pass `json` or `body`, not both. A GET or HEAD with a body throws. https://docs.nestjs.com/application/http-client#making-requests
- The type argument is not checked at runtime: validate responses from APIs you do not control. https://docs.nestjs.com/application/http-client#making-requests
- `responseType: 'auto'` parses JSON content types, returns a string otherwise, `undefined` for empty bodies. A 304 throws by default like any non-2xx. https://docs.nestjs.com/application/http-client#making-requests
- `headers` on the response is a web `Headers`: `res.headers.get('etag')`. https://docs.nestjs.com/application/http-client#making-requests

## Timeouts and cancellation

- No timeout by default (`0`). `timeout` applies to each attempt, not the whole call, so total time can reach attempts times timeout plus waits. For a whole-call deadline: `signal: AbortSignal.timeout(10_000)`. https://docs.nestjs.com/application/http-client#timeouts-and-cancellation
- An aborted request rejects with the signal's reason and is never retried. https://docs.nestjs.com/application/http-client#timeouts-and-cancellation

## Retries

- Retries are on by default: 3 attempts for GET, HEAD, OPTIONS, PUT, DELETE after connection errors, per-attempt timeouts and 408, 429, 500, 502, 503, 504, with full-jitter backoff and `Retry-After` honored. A stream body is never retried. https://docs.nestjs.com/application/http-client#retries
- POST and PATCH are not retried. Opt in per request with an idempotency key: https://docs.nestjs.com/application/http-client#retries
  ```typescript
  await this.payments.post('/charges', {
    json: charge,
    headers: { 'idempotency-key': `charge-${order.id}` },
    retry: { methods: ['POST'] },
  });
  ```
- `methods` and `statusCodes` replace the defaults (setting `methods: ['POST']` on a client stops GET retries). `retryIf` can only narrow. https://docs.nestjs.com/application/http-client#retries
- Retries multiply along service chains: retry at one layer, and set `retry: false` on clients calling your own services. https://docs.nestjs.com/application/http-client#retries

## Errors

- `HttpResponseError` (non-2xx), `HttpTimeoutError`, `HttpNetworkError`, `HttpParseError`, all extending `HttpClientError`. `throwOnHttpError: false` returns non-2xx as values. https://docs.nestjs.com/application/http-client#handling-errors
- An escaped `HttpClientError` is a 500. Map it with `toHttpException()` (502 for upstream and network errors, 504 for timeouts). The upstream body is not forwarded unless listed in `forward`. https://docs.nestjs.com/application/http-client#handling-errors
  ```typescript
  catch (error) {
    if (error instanceof HttpResponseError && error.status === 404) throw new NotFoundException();
    throw toHttpException(error);
  }
  ```
- `toHttpException()` logs every failure it turns into a 5xx. Pass `log: false` when your own filter logs. Add `DOMException` to a filter's `@Catch()` when using `AbortSignal.timeout()`. https://docs.nestjs.com/application/http-client#handling-errors
- In microservice or gateway handlers convert to `RpcException` or `WsException` yourself. https://docs.nestjs.com/application/http-client#handling-errors

## Interceptors, streaming, redirects, security

- Interceptors run once per attempt (`request.attempt`), in listed order, and see 4xx/5xx responses before the status check. Class interceptors can inject providers. https://docs.nestjs.com/application/http-client#interceptors
- `responseType: 'stream'` returns a Node `Readable` for `StreamableFile`. Do not forward upstream `content-length` (fetch decompresses). https://docs.nestjs.com/application/http-client#streaming-responses
- Proxy, pool and TLS go through an undici `dispatcher` (`new Agent()`, `new ProxyAgent()`). Install `undici` yourself. A Node `http.Agent` does not work. https://docs.nestjs.com/application/http-client#proxies-tls-and-redirects
- A client with `baseUrl` follows only same-origin redirects and refuses absolute URLs to other hosts, so its credentials never leak. Use a client without `baseUrl` for arbitrary hosts. https://docs.nestjs.com/application/http-client#security
- URLs with credentials and header values with line breaks throw. https://docs.nestjs.com/application/http-client#security

## Testing

- Stub with `HttpClientModule.forRoot({ fetch })` and give each call its own `Response` (`mockResolvedValueOnce(Response.json(...))`): a body reads once, and a retry would get a consumed one. https://docs.nestjs.com/application/http-client#testing
- E2E: `overrideProvider(HTTP_CLIENT_MODULE_OPTIONS).useValue({ fetch })` or `overrideProvider(getHttpClientToken('github'))`. https://docs.nestjs.com/application/http-client#testing

## Old patterns: @nestjs/axios

| `@nestjs/axios` | `@nestjs/http-client` |
| --- | --- |
| `firstValueFrom(http.get(url))` | `await http.get(url)` |
| `baseURL` | `baseUrl` |
| `post(url, data)` | `post(url, { json })` |
| `params` | `query` (arrays repeat the key) |
| `response.headers['etag']` | `response.headers.get('etag')` |
| `AxiosError` / `ECONNABORTED` | `HttpResponseError` / `HttpTimeoutError` |
| `validateStatus: () => true` | `throwOnHttpError: false` |
| `httpsAgent`, `proxy` | `dispatcher` |
| `maxRedirects: 0` | `redirect: 'manual'` |
| no retries, whole-request timeout | retries on (`retry: false` restores), per-attempt timeout |

https://docs.nestjs.com/application/http-client#migrating-from-nestjsaxios

- Both packages can be installed side by side while migrating. https://docs.nestjs.com/application/http-client
