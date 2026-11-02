# Migrations and wire protocol (Socket.IO 4.8)

## Contents

- Compatibility between majors
- Rolling upgrade from v2
- v2 to v3: defaults
- v2 to v3: CORS, cookies, auth
- v2 to v3: removed and changed APIs
- v3 to v4
- Engine.IO protocol
- Socket.IO protocol

## Compatibility between majors

- v3 and v4 share the protocol: a v3 client reaches a v4 server and the reverse. The 3 to 4 changes touch the server API only. https://socket.io/docs/v4/migrating-from-3-x-to-4-0/
- A v2 client reaches a v3 or v4 server only with `allowEIO3: true`. A v3 client cannot reach a v2 server (symptom `packet.data is undefined`). https://socket.io/docs/v4/migrating-from-2-x-to-3-0/
- Socket.IO v3 and later speak Socket.IO protocol 5 over Engine.IO protocol 4 (`EIO=4`). v1 and v2 use `EIO=3`. https://socket.io/docs/v4/socket-io-protocol/

## Rolling upgrade from v2

1. Upgrade the servers with `allowEIO3: true`.
2. Upgrade the clients, which can take time because of cached bundles.
3. Set `allowEIO3` back to `false`.

https://socket.io/docs/v4/migrating-from-2-x-to-3-0/

- `socket.conn.protocol` (3 or 4) tells which protocol a connection uses. https://socket.io/docs/v4/migrating-from-2-x-to-3-0/
- `socket.io-redis` 5 (with v2) and 6 (with v3) interoperate, so servers can move one by one. https://socket.io/docs/v4/migrating-from-2-x-to-3-0/

## v2 to v3: defaults

- `maxHttpBufferSize` drops from 100 MB to 1 MB: larger packets disconnect the client. https://socket.io/docs/v4/migrating-from-2-x-to-3-0/
- `perMessageDeflate` is off, CORS is off, `withCredentials` is `false`, no `io` cookie is sent. https://socket.io/docs/v4/migrating-from-2-x-to-3-0/
- `pingTimeout` is `20000` since v4. It had been lowered from `60000` to `5000` in `socket.io@2.1.0`, and the `5000` value caused disconnections with large payloads or a blocked event loop. https://socket.io/docs/v4/migrating-from-3-x-to-4-0/#the-default-value-of-pingtimeout-was-increased

## v2 to v3: CORS, cookies, auth

- `origins` and `handlePreflightRequest` become `cors`, passed to the `cors` package. https://socket.io/docs/v4/migrating-from-2-x-to-3-0/#cors-handling

```js
const io = new Server(httpServer, {
  cors: {
    origin: "https://example.com",
    methods: ["GET", "POST"],
    allowedHeaders: ["my-custom-header"],
    credentials: true,
  },
});
```

https://socket.io/docs/v4/migrating-from-2-x-to-3-0/#cors-handling

- `cookieName`, `cookieHttpOnly` and `cookiePath` become `cookie: { name, httpOnly, path }`. The cookie only matters for cookie-based sticky sessions with long-polling. https://socket.io/docs/v4/migrating-from-2-x-to-3-0/#no-more-cookie-by-default
- The Socket `query` option becomes `auth`, sent in the CONNECT packet. A reused Manager used to send the first namespace's token to the next one. The Manager `query` option remains for HTTP query parameters. https://socket.io/docs/v4/migrating-from-2-x-to-3-0/

```js
const socket = io({ auth: { token: "abc" } });
io.on("connection", (socket) => {
  console.log(socket.handshake.auth.token);
});
```

https://socket.io/docs/v4/migrating-from-2-x-to-3-0/

## v2 to v3: removed and changed APIs

| v2 | v3 and later |
| -- | ------------ |
| `io.set("authorization", fn)` | `io.use((socket, next) => ...)` |
| Connecting to `/admin` also ran `/` middlewares | Only `/admin` handlers run |
| `Namespace.connected` | `Namespace.sockets`, a `Map` |
| `socket.rooms` object | A `Set` |
| `socket.binary(false)` | A custom `parser` |
| `socket.join(room, cb)` | Synchronous, or a Promise with some adapters |
| `Namespace.clients(cb)` | `allSockets()`, a Promise of socket IDs (`fetchSockets()` arrives in v4) |
| Client `error` for middleware errors | `connect_error`, a real `Error` with `data` |
| Reconnection events on the Socket | On the Manager, `socket.io.on(...)` |
| Client `pong` for latency | An ack-based `ping` |
| `socket.emit(...).emit(...)` | `emit()` returns `true` |
| Default import | `import { Server } from "socket.io"`, `import { io } from "socket.io-client"` |
| `@types/socket.io*` | Bundled types |

Source: https://socket.io/docs/v4/migrating-from-2-x-to-3-0/

- Room names are no longer coerced: `socket.join(42)` does not match `io.to("42")`. https://socket.io/docs/v4/migrating-from-2-x-to-3-0/
- Latency with an ack: https://socket.io/docs/v4/migrating-from-2-x-to-3-0/#no-more-pong-event-for-retrieving-latency

```js
// server
socket.on("ping", (cb) => {
  if (typeof cb === "function") cb();
});

// client
setInterval(() => {
  const start = Date.now();
  socket.volatile.emit("ping", () => {
    const latency = Date.now() - start;
  });
}, 5000);
```

https://socket.io/docs/v4/migrating-from-2-x-to-3-0/#no-more-pong-event-for-retrieving-latency

## v3 to v4

- Broadcast operators are immutable. Building rooms with repeated `io.to()` and then calling `io.emit()` now emits to everyone: chain on 1 operator. https://socket.io/docs/v4/migrating-from-3-x-to-4-0/#ioto-is-now-immutable

```js
const operator = io.to("room1").to("room2");
operator.emit("hello"); // room1 or room2
```

https://socket.io/docs/v4/migrating-from-3-x-to-4-0/#ioto-is-now-immutable

- `socket.to("room1").broadcast.emit()` throws `cannot get emit of undefined`: `to()` already broadcasts. Use `socket.to("room1").emit()`. https://socket.io/docs/v4/migrating-from-3-x-to-4-0/#known-migration-issues
- `wsEngine` takes the module: `wsEngine: require("eiows").Server`. https://socket.io/docs/v4/migrating-from-3-x-to-4-0/#wsengine-option
- New in v4: `except()`, arrays in `to()`, `socketsJoin`, `socketsLeave`, `disconnectSockets`, `fetchSockets`, typed events, `autoUnref`. https://socket.io/docs/v4/migrating-from-3-x-to-4-0/

## Engine.IO protocol

- Query: `EIO=4`, `transport`, and `sid` after the handshake. Socket.IO serves it under `/socket.io/`. https://socket.io/docs/v4/engine-io-protocol/#query-parameters
- Packet types: open 0, close 1, ping 2, pong 3, message 4, upgrade 5, noop 6. https://socket.io/docs/v4/engine-io-protocol/#protocol
- The open packet carries `sid`, `upgrades`, `pingInterval`, `pingTimeout` and `maxPayload`. https://socket.io/docs/v4/engine-io-protocol/#handshake
- The server sends pings, the client answers within `pingTimeout`. The client gives up after `pingInterval + pingTimeout` without a ping. https://socket.io/docs/v4/engine-io-protocol/#heartbeat
- Long-polling allows 1 active GET and 1 active POST per session, otherwise HTTP 400 and the session closes. Packets are joined by `\x1e`, binary is base64 with a `b` prefix. https://socket.io/docs/v4/engine-io-protocol/#sending-and-receiving-data
- Upgrade: the client opens a WebSocket with the same `sid`, sends `2probe`, gets `3probe`, then sends `5`. https://socket.io/docs/v4/engine-io-protocol/#upgrade

## Socket.IO protocol

- Packet types: CONNECT 0, DISCONNECT 1, EVENT 2, ACK 3, CONNECT_ERROR 4, BINARY_EVENT 5, BINARY_ACK 6. https://socket.io/docs/v4/socket-io-protocol/#exchange-protocol
- The client sends CONNECT for every namespace, `/` included, before anything else. The Socket.IO `sid` differs from the Engine.IO `sid`. https://socket.io/docs/v4/socket-io-protocol/#connection-to-a-namespace
- An EVENT payload is a non-empty array. Namespaces must not contain a comma with the default parser. https://socket.io/docs/v4/socket-io-protocol/#packet-encoding

```
42["hello","world"]                      EVENT on /, inside an Engine.IO message
0/admin,{"sid":"oSO0OpakMV_3jnilAAAA"}   CONNECT to /admin
212["foo"]                               EVENT with ack id 12
3/admin,13["bar"]                        ACK on /admin
51-["baz",{"_placeholder":true,"num":0}] BINARY_EVENT with 1 attachment
```

https://socket.io/docs/v4/socket-io-protocol/#packet-encoding
