# HTTP and HTTPS clients

## Contents

- Requests
- Responses and headers
- Agents and keep-alive
- HTTPS client options
- Built-in proxy support
- Old patterns

## Requests

- Always attach `req.on('error')`: DNS, TCP and parse errors are emitted there and thrown without a listener. https://nodejs.org/docs/latest/api/http.html#httprequestoptions-callback
- Always call `req.end()` with `http.request()`, even without a body. https://nodejs.org/docs/latest/api/http.html#httprequestoptions-callback
- Compute `Content-Length` with `Buffer.byteLength(body)`, never `body.length`. Setting it disables chunked encoding. https://nodejs.org/docs/latest/api/http.html#httprequestoptions-callback
- `timeout` and `req.setTimeout()` only emit `'timeout'`: call `req.destroy()` yourself. https://nodejs.org/docs/latest/api/http.html#httprequestoptions-callback
- Cancel with the `signal` option. It behaves like `destroy()` and emits an `'error'` with code `ABORT_ERR`. https://nodejs.org/docs/latest/api/http.html#httprequestoptions-callback
- Header values outside latin1 throw. Encode them per RFC 8187: https://nodejs.org/docs/latest/api/http.html#requestsetheadername-value

```js
request.setHeader('Content-Disposition', `attachment; filename*=utf-8''${encodeURIComponent(filename)}`);
```

- `httpValidation` (added in v26.3.0) defaults to `'strict'`. `'relaxed'` allows limited non-ASCII like Fetch, `'insecure'` equals `insecureHTTPParser`: avoid it. https://nodejs.org/docs/latest/api/http.html#httprequestoptions-callback
- An `Authorization` header overrides `auth`. When `url` and `options` are both given, `options` wins. An absolute URL in `path` becomes absolute-form, so keep `path`, `host` and `Host` consistent. https://nodejs.org/docs/latest/api/http.html#httprequestoptions-callback
- With `Expect: 100-continue`, set a timeout and a `'continue'` listener. https://nodejs.org/docs/latest/api/http.html#httprequestoptions-callback
- Event order on success: `socket`, `response`, `data`, `end`, `close`. A close before the response gives `error` ("socket hang up", `ECONNRESET`). https://nodejs.org/docs/latest/api/http.html#httprequestoptions-callback

## Responses and headers

- Always consume the response body (`res.resume()` at least), even on error paths. Until consumed, `'end'` never fires and memory grows. https://nodejs.org/docs/latest/api/http.html#class-httpclientrequest

```js
http.get(url, (res) => {
  if (res.statusCode !== 200) { res.resume(); return; }
  res.setEncoding('utf8');
  let raw = '';
  res.on('data', (c) => { raw += c; });
  res.on('end', () => { try { JSON.parse(raw); } catch (e) { console.error(e.message); } });
}).on('error', (e) => console.error(e.message));
```
https://nodejs.org/docs/latest/api/http.html#httpgetoptions-callback

- `res` emits `'error'` only if an `'error'` listener exists. https://nodejs.org/docs/latest/api/http.html#class-httpclientrequest
- `message.headers` keeps only the first value of single-value headers (`content-type`, `authorization`, `host`, `location`...). Use `message.headersDistinct` (always arrays) or `joinDuplicateHeaders: true`. `set-cookie` is always an array. https://nodejs.org/docs/latest/api/http.html#messageheaders , https://nodejs.org/docs/latest/api/http.html#messageheadersdistinct
- Maximum header size is 16 KiB by default (`maxHeaderSize`, `--max-http-header-size`). https://nodejs.org/docs/latest/api/http.html#httpmaxheadersize

## Agents and keep-alive

- `http.globalAgent` keeps sockets alive with a 5 s timeout since v19.0.0. `new Agent()` defaults to `keepAlive: false`, `maxSockets: Infinity`, `scheduling: 'lifo'`. https://nodejs.org/docs/latest/api/http.html#httpglobalagent , https://nodejs.org/docs/latest/api/http.html#new-agentoptions
- Retry idempotent requests when a reused socket was closed by the server at the same moment: https://nodejs.org/docs/latest/api/http.html#requestreusedsocket

```js
const req = http.get(url, { agent }, (res) => {}).on('error', (err) => {
  if (req.reusedSocket && err.code === 'ECONNRESET') retriableRequest();
});
```

- `agentKeepAliveTimeoutBuffer` (default 1000 ms, added v24.7.0) expires sockets that long before the server's `keep-alive: timeout`. https://nodejs.org/docs/latest/api/http.html#new-agentoptions
- On a keep-alive connection, responses match requests only by order. Use `agent: false` or a dedicated Agent for isolation. https://nodejs.org/docs/latest/api/http.html#response-ordering-with-connection-reuse
- `destroy()` an Agent you no longer need, especially with `keepAlive`. Treat `agent.sockets`, `freeSockets`, `requests` as read-only. https://nodejs.org/docs/latest/api/http.html#agentdestroy , https://nodejs.org/docs/latest/api/http.html#agentsockets

## HTTPS client options

- Defaults: port 443 and `https.globalAgent` (keep-alive, 5 s). `servername` (SNI) is the host name, or `''` for an IP address. `maxCachedSessions` defaults to 100. https://nodejs.org/docs/latest/api/https.html#new-agentoptions
- Agents are pooled by TLS options, so put custom `ca`, `cert`, `key`, `rejectUnauthorized`, `servername` on the Agent, or use `agent: false`. A `checkServerIdentity` given per request (not at Agent construction) makes the request ineligible for connection and TLS session reuse. https://nodejs.org/docs/latest/api/https.html#httpsrequestoptions-callback
- Pin a certificate in `checkServerIdentity`: call `tls.checkServerIdentity(host, cert)` first and return its error, then compare the SHA-256 of `cert.pubkey` or `cert.fingerprint256`, returning an `Error` on mismatch. https://nodejs.org/docs/latest/api/https.html#httpsrequestoptions-callback
- The agent `'keylog'` event emits SSLKEYLOGFILE lines. The lines allow decrypting captured TLS traffic: write them with `mode: 0o600` and use it for debugging only. https://nodejs.org/docs/latest/api/https.html#event-keylog
- `NODE_EXTRA_CA_CERTS` adds root CAs from a PEM file and is ignored when `ca` is set explicitly. https://nodejs.org/docs/latest/api/cli.html#node_extra_ca_certsfile

## Built-in proxy support

Active development (Stability 1.1), added v24.5.0. https://nodejs.org/docs/latest/api/http.html#built-in-proxy-support

- Enable with `NODE_USE_ENV_PROXY=1` or `--use-env-proxy`, or call `http.setGlobalProxyFromEnv()` (added v24.14.0) before any request. It reads `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY` (lowercase wins), and covers `http`, `https` and `fetch`. Unix-socket requests ignore it. https://nodejs.org/docs/latest/api/http.html#built-in-proxy-support

```console
NODE_USE_ENV_PROXY=1 HTTP_PROXY=http://proxy.example.com:8080 NO_PROXY=localhost,127.0.0.1 node client.js
```

- A proxy is not a privacy boundary: configure only trusted ones. A tunnel failure raises `ERR_PROXY_TUNNEL`. https://nodejs.org/docs/latest/api/http.html#proxy-security-considerations , https://nodejs.org/docs/latest/api/errors.html#err_proxy_tunnel

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `req.abort()`, `'abort'`, `req.aborted` | `req.destroy()` or `signal`, listen to `'close'` | https://nodejs.org/docs/latest/api/http.html#event-abort |
| `message.aborted`, `'aborted'` event | `'close'` and `message.destroyed` | https://nodejs.org/docs/latest/api/http.html#messageaborted |
| `req.connection` | `req.socket` | https://nodejs.org/docs/latest/api/http.html#requestconnection |
| `req.finished` | `req.writableEnded` | https://nodejs.org/docs/latest/api/http.html#requestfinished |
| `OutgoingMessage.flush()` (removed, DEP0001) | `flushHeaders()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0001 |
| `clientCertEngine` (depends on OpenSSL engines, deprecated in OpenSSL 3) | `cert` and `key` options | https://nodejs.org/docs/latest/api/https.html#httpsrequestoptions-callback |
| `require('node:_http_agent')` and other `_http_*` | `node:http` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0199 |
