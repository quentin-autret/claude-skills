# HTTP and HTTPS servers

## Contents

- Timeouts and limits
- Graceful shutdown
- Requests
- Responses
- HTTPS servers
- Version-flagged additions
- Old patterns

## Timeouts and limits

- Keep `headersTimeout` (default min(`requestTimeout`, 60000)) and `requestTimeout` (default 300000 ms since v18) non-zero when no reverse proxy is in front: they guard against slow-client DoS. Both answer 408 without calling the handler. https://nodejs.org/docs/latest/api/http.html#serverheaderstimeout , https://nodejs.org/docs/latest/api/http.html#serverrequesttimeout
- `server.timeout` (socket inactivity) defaults to 0, no timeout. Once a `'timeout'` listener exists, timeouts must be handled explicitly (destroy the socket yourself). https://nodejs.org/docs/latest/api/http.html#servertimeout
- Set `keepAliveTimeout` explicitly: the property docs say 5000 ms while the `createServer` option lists 65000 ms. 0 disables it. https://nodejs.org/docs/latest/api/http.html#serverkeepalivetimeout , https://nodejs.org/docs/latest/api/http.html#httpcreateserveroptions-requestlistener
- `keepAliveTimeoutBuffer` (default 1000 ms, added v24.6.0) is added to the socket timeout to avoid `ECONNRESET` races with clients reusing sockets. https://nodejs.org/docs/latest/api/http.html#serverkeepalivetimeoutbuffer
- `maxHeadersCount` defaults to 1000. `maxRequestsPerSocket` defaults to 0 (no limit), and at the limit the response sets `Connection: close` without closing the socket, and later requests on it get 503. https://nodejs.org/docs/latest/api/http.html#servermaxheaderscount , https://nodejs.org/docs/latest/api/http.html#servermaxrequestspersocket
- Defaults: `requireHostHeader: true` (400 on HTTP/1.1 without Host), `noDelay: true`, `connectionsCheckingInterval: 30000`. https://nodejs.org/docs/latest/api/http.html#httpcreateserveroptions-requestlistener

## Graceful shutdown

- `server.close()` stops accepting, closes idle keep-alive connections (since v19) and waits for active ones. `closeIdleConnections()` is then redundant. https://nodejs.org/docs/latest/api/http.html#serverclosecallback
- To force, call `closeAllConnections()` after `close()`. It does not close upgraded sockets (WebSocket, HTTP/2). https://nodejs.org/docs/latest/api/http.html#servercloseallconnections

```js
server.close(() => console.log('closed'));
server.closeAllConnections();
```

- Prefer `await using server = http.createServer(handler)`: `[Symbol.asyncDispose]()` is `close()` as a promise (stable since v24.2.0). https://nodejs.org/docs/latest/api/http.html#serversymbolasyncdispose

## Requests

- `req.url` is only the path and query. Parse it against a configured origin, never against an unvalidated `Host` header. https://nodejs.org/docs/latest/api/http.html#messageurl

```js
const url = new URL(`http://${process.env.HOST ?? 'localhost'}${req.url}`);
```

- Cancel downstream work when the client disconnects with `req.signal` (added in v26.1.0): https://nodejs.org/docs/latest/api/http.html#messagesignal

```js
try {
  const data = await fetch('https://example.com/api', { signal: req.signal });
} catch (err) {
  if (err.name === 'AbortError') return;
  throw err;
}
```

- In `'clientError'` there is no `req`/`res`: write a raw HTTP response and close the socket before returning. Skip when `ECONNRESET` or not writable. https://nodejs.org/docs/latest/api/http.html#event-clienterror

```js
server.on('clientError', (err, socket) => {
  if (err.code === 'ECONNRESET' || !socket.writable) return;
  socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
});
```

- `optimizeEmptyRequests` (added v24.12.0, default false) gives bodiless requests an already-ended stream that never emits `data`/`end`: test `req.readableEnded`. https://nodejs.org/docs/latest/api/http.html#httpcreateserveroptions-requestlistener
- `shouldUpgradeCallback(req)` (added v24.9.0) decides which upgrades are accepted. https://nodejs.org/docs/latest/api/http.html#httpcreateserveroptions-requestlistener

## Responses

- Call `writeHead()` once, before `end()`. Its headers take precedence over merged `setHeader()` ones. If `setHeader()` was never called, they are written directly and `getHeader()` will not return them. The array form is flat `[k, v, k, v]`. https://nodejs.org/docs/latest/api/http.html#responsewriteheadstatuscode-statusmessage-headers

```js
res.writeHead(200, { 'Content-Length': Buffer.byteLength(body), 'Content-Type': 'text/plain' }).end(body);
```

- `strictContentLength: true` throws `ERR_HTTP_CONTENT_LENGTH_MISMATCH` on a wrong length. `rejectNonStandardBodyWrites` throws when writing a body that must not exist. https://nodejs.org/docs/latest/api/http.html#responsestrictcontentlength , https://nodejs.org/docs/latest/api/http.html#httpcreateserveroptions-requestlistener
- Send preload hints with `res.writeEarlyHints({ link: ['</styles.css>; rel=preload; as=style'] })`. https://nodejs.org/docs/latest/api/http.html#responsewriteearlyhintshints-callback
- Keep `sendDate` on except in tests: most responses require `Date`. https://nodejs.org/docs/latest/api/http.html#responsesenddate

## HTTPS servers

- `https.createServer()` accepts the options of `tls.createServer()`, `tls.createSecureContext()` and `http.createServer()`. https://nodejs.org/docs/latest/api/https.html#httpscreateserveroptions-requestlistener
- Dev certificate: `openssl req -x509 -newkey rsa:2048 -nodes -sha256 -subj '/CN=localhost' -keyout private-key.pem -out certificate.pem`. https://nodejs.org/docs/latest/api/https.html#httpscreateserveroptions-requestlistener
- Client certificate details come from `req.socket.getPeerCertificate()`. https://nodejs.org/docs/latest/api/http.html#messagesocket
- The https page lists `server.setTimeout()` with default 120000 while `server.timeout` says 0 (changed in v13.0.0). Treat 0 as the default and set what you need. https://nodejs.org/docs/latest/api/https.html#serversettimeoutmsecs-callback , https://nodejs.org/docs/latest/api/https.html#servertimeout

## Version-flagged additions

| API | Added | Source |
| :-- | :-- | :-- |
| `res.writeInformation(status, headers)` for any 1xx except 101 | v26.2.0 | https://nodejs.org/docs/latest/api/http.html#responsewriteinformationstatuscode-headers-callback |
| `http.isValidHeaderName(name)` | v26.11.0 | https://nodejs.org/docs/latest/api/http.html#httpisvalidheadernamename |
| `http.websocketMask()` / `websocketUnmask()` (RFC 6455 XOR masking) | v26.12.0 | https://nodejs.org/docs/latest/api/http.html#httpwebsocketmasksource-mask-output-offset-length |
| `req.signal` (no longer aborted after normal completion since v26.7.0) | v26.1.0 | https://nodejs.org/docs/latest/api/http.html#messagesignal |

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `res.finished` | `res.writableEnded` | https://nodejs.org/docs/latest/api/http.html#responsefinished |
| `res.connection` | `res.socket` | https://nodejs.org/docs/latest/api/http.html#responseconnection |
| `server.closeIdleConnections()` before `close()` | `close()` alone (v19+) | https://nodejs.org/docs/latest/api/http.html#servercloseidleconnections |
| `server.listen({ signal })` to shut down | `await using server = ...` or `close()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0209 |
| `http.Server()` without `new` | `new http.Server()` or `http.createServer()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0195 |
| `url.parse(req.url)` | `new URL(req.url, origin)` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0169 |
