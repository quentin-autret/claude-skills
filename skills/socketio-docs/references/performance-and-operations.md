# Performance and operations (Socket.IO 4.8)

## Contents

- WebSocket server
- Memory
- Custom parsers
- OS limits
- Load testing
- Admin UI
- Debug logs

## WebSocket server

- Performance follows the WebSocket server, `ws` by default. Install its native add-ons, or it falls back to JavaScript. https://socket.io/docs/v4/performance-tuning/#install-ws-native-add-ons

```sh
npm install --save-optional bufferutil utf-8-validate
```

- Swap the engine with `wsEngine`, which takes a WebSocket server class (any implementation exposing the same API as `ws`, notably `handleUpgrade`). `eiows` "allows, but doesn't guarantee" gains, so benchmark it. https://socket.io/docs/v4/server-installation/#other-websocket-server-implementations

```js
const io = new Server(3000, { wsEngine: require("eiows").Server });
```

https://socket.io/docs/v4/performance-tuning/#use-another-websocket-server-implementation

- Keep `perMessageDeflate` off unless needed: it costs CPU and memory. https://socket.io/docs/v4/server-options/#permessagedeflate

## Memory

- Memory scales linearly with the number of connected clients. Resource use also depends on the number of messages (emits, emits with acknowledgement and broadcasts) received and sent per second. https://socket.io/docs/v4/memory-usage/
- Each session keeps its first HTTP request for `express-session`. Discard it when unused. https://socket.io/docs/v4/performance-tuning/#discard-the-initial-http-request

```js
io.engine.on("connection", (rawSocket) => {
  rawSocket.request = null;
});
```

https://socket.io/docs/v4/performance-tuning/#discard-the-initial-http-request

- Dynamic namespaces each hold an adapter. Remove empty ones with `cleanupEmptyChildNamespaces: true`. https://socket.io/docs/v4/server-options/#cleanupemptychildnamespaces
- Avoid tiny `pingInterval` values: `1000` (one heartbeat per second) adds load that can become noticeable with a few thousand connected clients. https://socket.io/docs/v4/server-options/#pinginterval

## Custom parsers

- The default parser sends each binary attachment in its own WebSocket frame. A msgpack parser sends binary packets in 1 frame and can shrink number-heavy payloads, but needs `ArrayBuffer` support and is harder to read in the Network tab. https://socket.io/docs/v4/custom-parser/#the-msgpack-parser
- Pass the `parser` option on both the server and the client. https://socket.io/docs/v4/custom-parser/

```js
// server
const parser = require("socket.io-msgpack-parser");
const io = new Server({ parser });

// client
const socket = io(SERVER_URL, { parser });
```

https://socket.io/docs/v4/custom-parser/

- In a browser, the official `socket.io.msgpack.min.js` bundle already includes the msgpack parser, with no `parser` option. https://socket.io/docs/v4/custom-parser/#the-msgpack-parser
- Other parsers: `socket.io-circular-parser` (circular references), `@skgdev/socket.io-msgpack-javascript` (msgpack based on the official JavaScript implementation, while `socket.io-msgpack-parser` uses `notepack.io`, which lacks extension types), `socket.io-json-parser`, `socket.io-cbor-x-parser`, `@socket.io/devalue-parser`. https://socket.io/docs/v4/custom-parser/#available-parsers
- A parser exports `{ Encoder, Decoder }`. `Encoder.encode(packet)` returns an array of strings or buffers. `Decoder.add(chunk)` emits `"decoded"` and throws on invalid input, `destroy()` frees buffers. Validate the type (0 to 4), that `nsp` is a string and that `id` is an integer or undefined. https://socket.io/docs/v4/custom-parser/#implementing-your-own-parser

## OS limits

- Stuck near 1000 connections: the open files limit (`ulimit -n` shows 1024). Raise `nofile` in `/etc/security/limits.d/`. https://socket.io/docs/v4/performance-tuning/#at-the-os-level
- Stuck near 28000 connections: the local port range. Set `net.ipv4.ip_local_port_range = 10000 65535` and reboot, which allows about 55000 connections per incoming IP. https://socket.io/docs/v4/performance-tuning/#at-the-os-level

## Load testing

- Generate load with the Socket.IO client, since the protocol has its own handshake, heartbeat and encoding. https://socket.io/docs/v4/load-testing/
- Artillery's default engine ships a v2 client that cannot reach a v4 server. Use `artillery-engine-socketio-v3`. https://socket.io/docs/v4/load-testing/#artillery

```yaml
config:
  target: "http://localhost:3000"
  engines:
    socketio-v3: {}
```

https://socket.io/docs/v4/load-testing/#artillery

- Artillery cannot easily test server-to-client events. For those, write a script: create clients every 10 ms, restrict a share to `["polling"]`, log `disconnect` reasons and count packets per second. https://socket.io/docs/v4/load-testing/#manual-client-creation

```js
const transports = Math.random() < POLLING_PERCENTAGE ? ["polling"] : ["polling", "websocket"];
const socket = io(URL, { transports });
```

https://socket.io/docs/v4/load-testing/#manual-client-creation

## Admin UI

- `@socket.io/admin-ui`: call `instrument(io, { auth, mode })`. The hosted UI at admin.socket.io needs CORS on the server. https://socket.io/docs/v4/admin-ui/#server-side

```js
const { instrument } = require("@socket.io/admin-ui");
const io = new Server(httpServer, {
  cors: { origin: ["https://admin.socket.io"], credentials: true },
});
instrument(io, {
  auth: { type: "basic", username: "admin", password: "$2b$10$..." }, // bcrypt hash
  mode: "production",
});
```

https://socket.io/docs/v4/admin-ui/#server-side

- `auth` is mandatory. `auth: false` disables authentication. https://socket.io/docs/v4/admin-ui/#auth
- `bcrypt` rejects hashes starting with `$2y$`. Generate one with `require("bcryptjs").hashSync("changeit", 10)`. https://socket.io/docs/v4/admin-ui/#auth
- `mode` defaults to `development`, which sends every socket and room detail. Set `"production"` to cut memory. https://socket.io/docs/v4/admin-ui/#mode
- With several servers, give each a distinct `serverId` when they share a machine, and a shared `store: new RedisStore(redisClient)` so basic-auth sessions survive a reconnection to another server. https://socket.io/docs/v4/admin-ui/#available-options
- `namespaceName` defaults to `/admin`, a regular namespace reachable with `io.of("/admin")`. `readonly` defaults to `false`. https://socket.io/docs/v4/admin-ui/#available-options
- In NestJS, call `instrument(this.server, ...)` in the gateway's `afterInit()`. https://socket.io/docs/v4/admin-ui/#server-side

## Debug logs

- Socket.IO is silent by default and logs through `debug`. Node.js: `DEBUG=socket.io* node app.js`, or `DEBUG=engine,socket.io*`. Browser: `localStorage.debug = '*'`. https://socket.io/docs/v4/logging-and-debugging/#available-debugging-scopes
- `debug` adds about 4 KB (minified and gzipped) to browser bundles and is excluded from the slim bundle. With webpack, remove it from other bundles with `webpack-remove-debug`. https://socket.io/docs/v4/logging-and-debugging/#removing-debug-from-your-browser-bundle
- `net::ERR_CONNECTION_REFUSED`, `WebSocket is already in CLOSING or CLOSED state` and similar console lines come from the browser, not from the library. https://socket.io/docs/v4/logging-and-debugging/#error-logs-in-the-browser-console
