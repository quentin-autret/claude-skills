# Client (socket.io-client 4.8)

## Contents

- Installation and bundles
- Initialization
- Socket state and events
- connect_error and disconnect
- Offline behavior
- Emitting and listening
- Manager and multiplexing
- Connection options
- Reconnection options
- Socket options
- Bundlers

## Installation and bundles

- JS clients 3.x and 4.x work with servers 3.x and 4.x. A 2.x client needs `allowEIO3: true` on the server. https://socket.io/docs/v4/client-installation/#version-compatibility
- The server serves a bundle at `/socket.io/socket.io.js` that defines a global `io`. https://socket.io/docs/v4/client-installation/#standalone-build
- CDN bundles: `socket.io.js` (with `debug`, for development), `socket.io.min.js` (for production), `socket.io.msgpack.min.js` (with the msgpack parser). The documented CDN snippet carries `integrity` and `crossorigin="anonymous"` attributes. https://socket.io/docs/v4/client-installation/#from-a-cdn
- With npm, the package works with bundlers and can also run from Node.js. Its types are bundled, do not add `@types/socket.io-client`. https://socket.io/docs/v4/client-installation/#from-npm
- Without the standalone build, import `io` from `socket.io-client` (build tools, Node.js or React Native). https://socket.io/docs/v4/client-api/#io

## Initialization

- `io()` connects to `window.location`. Cross-origin, pass the URL and enable CORS on the server. https://socket.io/docs/v4/client-initialization/#from-a-different-domain
- `http` and `ws`, `https` and `wss` are equivalent. A URL without scheme works only in a browser page served over HTTPS. https://socket.io/docs/v4/client-initialization/#from-a-different-domain
- The URL pathname is the namespace: `io("/admin")` or `io("https://example.com/admin")`. The request path is the `path` option. https://socket.io/docs/v4/client-initialization/#custom-namespace

```js
const socket = io("https://example.com/order", { path: "/my-custom-path/" });
// namespace "/order", requests to /my-custom-path/?EIO=4&transport=polling
```

https://socket.io/docs/v4/client-options/#path

## Socket state and events

- `socket.id` is `undefined` before `connect` and after `disconnect`, and is regenerated after each reconnection (unless connection state recovery is enabled). Do not use it as an identity. https://socket.io/docs/v4/client-socket-instance/#socketid
- `socket.io` is the Manager, `socket.io.engine` the Engine.IO client. `socket.io.engine.transport.name` is usually `polling` at `connect`, then `websocket`. https://socket.io/docs/v4/client-socket-instance/#socketio

- The Socket emits `connect`, `connect_error` and `disconnect`. Reconnection events are on the Manager: `reconnect_attempt`, `reconnect`, `reconnect_error`, `reconnect_failed`, `error`, `ping`. https://socket.io/docs/v4/client-api/#events

```js
socket.io.on("reconnect_attempt", () => { /* ... */ });
socket.io.on("reconnect", () => { /* ... */ });
```

- `connect` fires on every reconnection. Register other listeners outside it. https://socket.io/docs/v4/client-socket-instance/#connect
- `socket.recovered` tells whether the last reconnection restored the session. If `true`, missed events arrive now. https://socket.io/docs/v4/client-api/#socketrecovered

## connect_error and disconnect

- `socket.active` separates the cases: `true` is a temporary failure that will be retried, `false` is a denial by a server middleware, and `socket.connect()` must be called manually. https://socket.io/docs/v4/client-socket-instance/#connect_error

```js
socket.on("connect_error", (error) => {
  if (socket.active) {
    // temporary failure, the socket will retry
  } else {
    // denied by the server, socket.connect() must be called manually
    console.log(error.message);
  }
});
```

https://socket.io/docs/v4/client-socket-instance/#connect_error

- `connect_error` carries `err.message` (such as "xhr poll error"), `err.description` and `err.context`. https://socket.io/docs/v4/troubleshooting-connection-issues/#problem-the-socket-is-not-able-to-connect
- `disconnect` receives `(reason, details)`. https://socket.io/docs/v4/client-socket-instance/#disconnect

| `disconnect` reason | Auto-reconnect |
| ------------------- | -------------- |
| `io server disconnect` (server called `socket.disconnect()`) | No |
| `io client disconnect` (client called `socket.disconnect()`) | No |
| `ping timeout` | Yes |
| `transport close` | Yes |
| `transport error` | Yes |

## Offline behavior

- Emits made while disconnected are buffered and flushed on reconnection, which can cause a spike. Check `socket.connected` or use `socket.volatile.emit(...)`. https://socket.io/docs/v4/client-offline-behavior/#buffered-events

## Emitting and listening

- With `timeout(ms)` the ack callback takes `(err, value)`, without it `(value)`. https://socket.io/docs/v4/client-api/#socketemitwithackeventname-args
- `socket.off(event, listener)` needs the same function reference. `socket.off(event)` removes all listeners of the event, `socket.off()` all listeners. https://socket.io/docs/v4/client-api/#socketoffeventname-listener
- `socket.disconnect()` stops reconnection. It closes the low-level connection only if no other Socket uses the Manager. https://socket.io/docs/v4/client-api/#socketdisconnect

## Manager and multiplexing

- Sockets of different namespaces on the same host share 1 Manager and 1 connection. `forceNew: true` or `multiplex: false` creates a new Manager. Calling `io()` twice for the same namespace also does. https://socket.io/docs/v4/client-options/#forcenew
- `manager.socket(nsp, options)` only reads `auth`. Connection options go to the Manager. https://socket.io/docs/v4/client-api/#managersocketnsp-options

```js
const manager = new Manager("https://example.com", {
  reconnectionDelayMax: 10000,
  transports: ["websocket"],
});
const adminSocket = manager.socket("/admin", { auth: { token: "123" } });
```

https://socket.io/docs/v4/client-api/#managersocketnsp-options

## Connection options

- `transports`: default `["polling", "websocket", "webtransport"]`. `["websocket"]` removes long-polling and the need for sticky sessions. https://socket.io/docs/v4/client-options/#transports
- To try WebSocket first, reverse the order and restore it on error. https://socket.io/docs/v4/client-options/#transports

```js
const socket = io("https://example.com", { transports: ["websocket", "polling"] });
socket.on("connect_error", () => {
  socket.io.opts.transports = ["polling", "websocket"];
});
```

https://socket.io/docs/v4/client-options/#transports

- `tryAllTransports` (default `false`) tests the other transports when the first fails, at the cost of slower failure detection. https://socket.io/docs/v4/client-options/#tryalltransports
- `rememberUpgrade` (default `false`) starts with WebSocket when the last one succeeded. Use it only over TLS or networks known not to block WebSocket. https://socket.io/docs/v4/client-options/#rememberupgrade
- `path` (default `/socket.io/`) must match the server value, unless a path-rewriting proxy sits in between. https://socket.io/docs/v4/client-options/#path
- `addTrailingSlash` (default `true`, added in 4.6.0): with `false`, the request URL is `/socket.io` instead of `/socket.io/`. https://socket.io/docs/v4/client-options/#addtrailingslash
- `query` lands in `socket.handshake.query` and only changes for the next session (update `socket.io.opts.query` in `reconnect_attempt`). `EIO`, `transport`, `sid`, `j` and `t` are reserved. https://socket.io/docs/v4/client-options/#query

- `extraHeaders` reach `socket.handshake.headers`, except in a browser with only WebSocket enabled. https://socket.io/docs/v4/client-options/#extraheaders
- `withCredentials` (default `false`) sends cookies cross-origin. The server needs `cors.credentials: true` and explicit origins. https://socket.io/docs/v4/client-options/#withcredentials
- `closeOnBeforeunload` (default `false`): with `false`, only Firefox emits `disconnect` on reload. Keep `false` when the page uses a `beforeunload` prompt. https://socket.io/docs/v4/client-options/#closeonbeforeunload
- Node.js only: `agent`, `pfx`, `key`, `passphrase`, `cert`, `ca`, `ciphers`, `rejectUnauthorized`. `rejectUnauthorized` does not bypass the security check in a browser. https://socket.io/docs/v4/client-options/#nodejs-specific-options

## Reconnection options

- `autoConnect` (default `true`): with `false`, call `socket.connect()`. https://socket.io/docs/v4/client-options/#autoconnect
- `reconnection` (default `true`), `reconnectionAttempts` (`Infinity`), `reconnectionDelay` (1000 ms), `reconnectionDelayMax` (5000 ms), `randomizationFactor` (0.5). The delay doubles each attempt: first between 500 and 1500 ms, second between 1000 and 3000 ms. https://socket.io/docs/v4/client-options/#randomizationfactor
- `timeout` (default 20000 ms) bounds each connection attempt. It is not `ackTimeout`. https://socket.io/docs/v4/client-options/#timeout

## Socket options

- `auth`: object or `(cb) => cb({...})`, read as `socket.handshake.auth`. https://socket.io/docs/v4/client-options/#auth
- `ackTimeout` and `retries` go together: the client resends an unacknowledged event up to `retries + 1` times, then drops it. The server must acknowledge it. https://socket.io/docs/v4/client-options/#retries

```js
const socket = io({ retries: 3, ackTimeout: 10000 });
socket.emit("my-event");                                 // implicit ack
socket.timeout(5000).emit("my-event", (err, val) => {}); // custom timeout
```

https://socket.io/docs/v4/client-options/#retries

## Bundlers

- Webpack 5 for Node.js: `target: "node"` and `bufferutil`, `utf-8-validate` as externals, or `ReferenceError: document is not defined`. https://socket.io/docs/v4/client-with-bundlers/#nodejs
