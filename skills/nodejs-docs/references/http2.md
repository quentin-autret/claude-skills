# node:http2

## Contents

- Servers and ALPN
- Streams and sessions
- Headers
- Hardening options
- Old patterns

## Servers and ALPN

- Browsers do not speak unencrypted HTTP/2: use `http2.createSecureServer()`. `createServer()` is plain h2c. https://nodejs.org/docs/latest/api/http2.html#http2createserveroptions-onrequesthandler
- To serve HTTP/1.1 and HTTP/2 on one port, set `allowHTTP1: true` (default `false`) and use only the HTTP/1 public API in the handler. Upgrading from a non-TLS HTTP/1 server is not supported (compatibility API note). https://nodejs.org/docs/latest/api/http2.html#http2createsecureserveroptions-onrequesthandler

```js
createSecureServer({ cert, key, allowHTTP1: true }, onRequest).listen(8000);
function onRequest(req, res) {
  const { socket: { alpnProtocol } } = req.httpVersion === '2.0' ? req.stream.session : req;
}
```
https://nodejs.org/docs/latest/api/http2.html#alpn-negotiation

- `'unknownProtocol'` fires only when `allowHTTP1` is false and the client sent no ALPN. Without a listener the connection is terminated; with one, the server destroys the socket after `unknownProtocolTimeout` (10000 ms) if the handler has not. https://nodejs.org/docs/latest/api/http2.html#event-unknownprotocol
- The compatibility API `(req, res)` covers only the HTTP/1 public API: middleware relying on internals breaks, and status messages are ignored. https://nodejs.org/docs/latest/api/http2.html#compatibility-api
- `http1Options` (added in v25.7.0) configures the HTTP/1 fallback, including `keepAliveTimeout` (5000). https://nodejs.org/docs/latest/api/http2.html#http2createserveroptions-onrequesthandler

## Streams and sessions

- Attach `'error'` to every stream: a network error destroys each stream individually. https://nodejs.org/docs/latest/api/http2.html#event-stream
- Do not read or write the raw socket of an `Http2Session` (it can leave the session unusable): use the session API. https://nodejs.org/docs/latest/api/http2.html#http2session-and-sockets
- `respondWithFile()` needs a regular file, otherwise an `'error'` event is emitted on the stream. Use `statCheck` and `onError`. https://nodejs.org/docs/latest/api/http2.html#http2streamrespondwithfilepath-headers-options
- `pushStream()` from inside a pushed stream throws. Clients disable push with `session.settings({ enablePush: false })`. https://nodejs.org/docs/latest/api/http2.html#http2streampushstreamheaders-options-callback
- `setLocalWindowSize(n)` raises the window at once but lowering lags (an advertised window cannot be retracted). For a small initial window use `connectionWindowSize` (added in v26.11.0). On a client call it from `'connect'` or `'remoteSettings'`. https://nodejs.org/docs/latest/api/http2.html#http2sessionsetlocalwindowsizewindowsize
- Observe `entryTypes: ['http2']` with a `PerformanceObserver` for session and stream stats. https://nodejs.org/docs/latest/api/http2.html#collecting-http2-performance-metrics

## Headers

- Mark low-entropy secrets as never-indexed for HPACK: https://nodejs.org/docs/latest/api/http2.html#sensitive-headers

```js
const headers = {
  ':status': '200',
  'cookie': 'some-cookie',
  [http2.sensitiveHeaders]: ['cookie'],
};
```

- Build direct HTTP/2 requests with `:authority`. When converting from HTTP/1 the compat API falls back to `host`, raw `req.headers` does not. https://nodejs.org/docs/latest/api/http2.html#note-on-authority-and-host
- Text defaults to UTF-8: set the `content-type` charset. https://nodejs.org/docs/latest/api/http2.html#class-http2stream
- `strictSingleValueFields` (added in v25.7.0, default true) throws when a single-value header gets several values. `strictFieldWhitespaceValidation` defaults to true (RFC 9113). https://nodejs.org/docs/latest/api/http2.html#http2createserveroptions-onrequesthandler

## Hardening options

Defaults from https://nodejs.org/docs/latest/api/http2.html#http2createserveroptions-onrequesthandler

| Option | Default |
| :-- | :-- |
| `maxSessionMemory` | 10 (MB) |
| `maxHeaderListPairs` | 128 |
| `peerMaxConcurrentStreams` | 100 |
| `maxSessionRejectedStreams` | 100 |
| `maxSessionInvalidFrames` | 1000 |
| `maxOutstandingPings` / `maxSettings` | 10 / 32 |
| `streamResetBurst` / `streamResetRate` | 1000 / 33 (both must be set to take effect) |

- `streamResetBurst` and `streamResetRate` (added v22.10.0) rate-limit incoming RST_STREAM against rapid-reset DoS. https://nodejs.org/docs/latest/api/http2.html#http2createserveroptions-onrequesthandler
- Exceeding `maxSendHeaderBlockLength` emits `'frameError'` and destroys the stream. https://nodejs.org/docs/latest/api/http2.html#http2createserveroptions-onrequesthandler

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `stream.priority()`, `weight` option | nothing, priority signaling is removed (RFC 9113) | https://nodejs.org/docs/latest/api/http2.html#http2streampriorityoptions |
| `Http1IncomingMessage`, `Http1ServerResponse` options | `http1Options: { IncomingMessage, ServerResponse }` | https://nodejs.org/docs/latest/api/deprecations.html#dep0202-http1incomingmessage-and-http1serverresponse-options-of-http2-servers |
| `req.connection`, `res.connection` | `req.socket`, `res.socket` | https://nodejs.org/docs/latest/api/http2.html#requestconnection |
| `res.finished` | `res.writableEnded` | https://nodejs.org/docs/latest/api/http2.html#responsefinished |
