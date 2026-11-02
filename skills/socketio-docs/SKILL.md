---
name: socketio-docs
description: Builds, configures and debugs Socket.IO 4.8 servers and clients from the official Socket.IO v4 documentation. Use when writing a Node.js Socket.IO server (Express, Fastify, Koa, Bun, uWebSockets.js) or a socket.io-client app, emitting events with acknowledgements, timeout() and emitWithAck, broadcasting to rooms and namespaces, authenticating in middleware with the auth option, configuring CORS, reverse proxies and sticky sessions, scaling with the Redis, Redis Streams, Postgres, MongoDB or Cluster adapter, enabling connection state recovery and delivery guarantees, tuning reconnection, writing tests, troubleshooting connection issues, or migrating from Socket.IO v2 or v3. Triggers on Socket.IO, socket.io, socket.io-client, io(), new Server, emit, emitWithAck, acknowledgements, rooms, namespaces, broadcast, io.use, middleware, CORS, Redis adapter, sticky sessions, connection state recovery, reconnection, connect_error, Session ID unknown. Not for plain WebSocket servers such as ws without Socket.IO.
metadata:
  version: "4.8"
  source: https://socket.io/docs/v4/
  read: "2026-11-02"
---

# Socket.IO 4.8

## Overview

This skill holds the Socket.IO 4.8 rules that are easy to get wrong: an ephemeral `socket.id` used as a user key, listeners registered twice, acknowledgements whose callback shape changes with `timeout()`, CORS that does not protect WebSocket, sticky sessions forgotten behind a load balancer, and v2 or v3 APIs that no longer work. Every claim carries the page URL it comes from. The reference files hold the detail per domain.

## Version covered

Socket.IO 4.8 (server `socket.io`, client `socket.io-client`), docs v4. JS clients 3.x and 4.x talk to servers 3.x and 4.x. https://socket.io/docs/v4/client-installation/#version-compatibility

## Quick start

- Attach to the HTTP server you create, and call `listen` on it, never on the Express `app`: `app.listen()` creates another HTTP server that Socket.IO is not attached to. https://socket.io/docs/v4/server-initialization/#with-express

```js
import express from "express";
import { createServer } from "node:http";
import { Server } from "socket.io";

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, { cors: { origin: ["https://example.com"] } });

io.on("connection", (socket) => {
  socket.on("hello", (arg, callback) => callback("got it"));
});

httpServer.listen(3000);
```

https://socket.io/docs/v4/server-initialization/#with-express

- Client: `io()` on the same origin, `io("https://server-domain.com")` cross-origin (CORS must be enabled on the server). Import with `import { io } from "socket.io-client"`. https://socket.io/docs/v4/client-initialization/#from-a-different-domain

```js
import { io } from "socket.io-client";
const socket = io("https://server-domain.com", { auth: { token: "abc" } });
const answer = await socket.timeout(5000).emitWithAck("hello", "world");
```

https://socket.io/docs/v4/client-api/#socketemitwithackeventname-args

- Types ship with `socket.io-client`. Do not install `@types/socket.io-client`, it conflicts with the bundled typings. https://socket.io/docs/v4/client-installation/#from-npm

## Rules and pitfalls

### Connection and identity

- Socket.IO is not a WebSocket implementation: a plain WebSocket client cannot reach a Socket.IO server, and the Socket.IO client cannot reach a plain WebSocket server,. https://socket.io/docs/v4/#what-socketio-is-not
- Unless connection state recovery is enabled, `socket.id` changes on every reconnection and page refresh, differs per tab and has no message queue. Identify users with your own session ID sent in a cookie or in `auth`, and reach every tab of a user through a room named after the user ID. https://socket.io/docs/v4/server-socket-instance/#socketid

```js
io.on("connection", async (socket) => {
  const userId = await computeUserIdFromHeaders(socket.handshake.headers);
  socket.join(userId);
  io.to(userId).emit("hi");
});
```

https://socket.io/docs/v4/rooms/#sample-use-cases

- Authenticate in a connection middleware with `socket.handshake.auth`. Call `next()` in every code path, otherwise the connection is left hanging until it is closed after a timeout. `next(err)` reaches the client as `connect_error`, with `err.data` for details. https://socket.io/docs/v4/middlewares/

```js
io.use((socket, next) => {
  if (isValid(socket.handshake.auth.token)) return next();
  const err = new Error("not authorized");
  err.data = { content: "Please retry later" };
  next(err);
});
```

https://socket.io/docs/v4/middlewares/#handling-middleware-error

- A connection denied by a middleware is not retried. In `connect_error`, `socket.active === false` means the connection was denied by the server and the client must call `socket.connect()` itself to reconnect. https://socket.io/docs/v4/client-socket-instance/#connect_error
- Register client listeners at top level, never inside the `connect` handler: `connect` fires again on every reconnection and adds a duplicate listener each time. https://socket.io/docs/v4/client-socket-instance/#connect
- On the server, register `socket.on(...)` synchronously in the `connection` handler, before any `await`, or early packets reach no handler. https://socket.io/docs/v4/troubleshooting-connection-issues/#delayed-event-handler-registration
- Express middlewares go to `io.engine.use(...)`, not to `io.use(...)`. https://socket.io/docs/v4/middlewares/#compatibility-with-express-middleware

### Events and acknowledgements

- Pass objects as they are. Never `JSON.stringify()` a payload. `Date` arrives as a string, `Map` and `Set` must be converted by hand. https://socket.io/docs/v4/emitting-events/#basic-emit
- The ack callback is the last argument. Without `timeout()` it receives `(response)`, with `timeout(ms)` it receives `(err, response)`. https://socket.io/docs/v4/emitting-events/#with-timeout
- Prefer `await socket.timeout(ms).emitWithAck(...)` in a `try` block: it rejects on timeout. The receiving side calls its callback with 1 argument. https://socket.io/docs/v4/client-api/#socketemitwithackeventname-args
- A broadcast with acknowledgement requires `timeout()` and returns 1 response per client. https://socket.io/docs/v4/server-api/#serveremiteventname-args

```js
const responses = await io.timeout(10000).emitWithAck("some-event");
```

https://socket.io/docs/v4/server-api/#serveremitwithackeventname-args

- Listeners have no built-in error handling and no validation. Catch errors in each handler, validate payloads with a schema library and check `typeof callback === "function"` before using it. https://socket.io/docs/v4/listening-to-events/#validation
- Never use `connect`, `connect_error`, `disconnect`, `disconnecting`, `newListener` or `removeListener` as an application event: emitting a reserved name such as `socket.emit("disconnecting")` throws. https://socket.io/docs/v4/emit-cheatsheet/

### Broadcasting and rooms

- `socket.to(room)` and `socket.broadcast` exclude the sender, `io.to(room)` does not. `socket.to(socket.id).emit()` sends nothing: use `io.to(socket.id).emit()`. https://socket.io/docs/v4/server-api/#sockettoroom
- Chained `to()` calls are a union, each socket receives the event once. https://socket.io/docs/v4/rooms/#joining-and-leaving
- Each broadcast modifier returns a new immutable operator: keep it in a variable instead of calling `io.to()` and then `io.emit()`. https://socket.io/docs/v4/migrating-from-3-x-to-4-0/#ioto-is-now-immutable
- Room names are not coerced: `socket.join(42)` does not match `io.to("42")`. https://socket.io/docs/v4/migrating-from-2-x-to-3-0/#room-names-are-not-coerced-to-string-anymore
- Rooms are left automatically on disconnection. Read them in `disconnecting`, not in `disconnect` where `socket.rooms` is empty. https://socket.io/docs/v4/rooms/#disconnection
- Read the adapter with `io.of("/").adapter`: the docs warn that `io.adapter()` will not work for that, it is the setter. https://socket.io/docs/v4/adapter/#api

### Delivery

- The default guarantee is at most once and the server stores no message: a disconnected client misses broadcasts for good. Replay from a database with an offset sent in `auth`, or enable connection state recovery. https://socket.io/docs/v4/delivery-guarantees#at-most-once
- The client buffers emits while offline and flushes them on reconnection. Check `socket.connected` or use `socket.volatile.emit(...)` for data that can be dropped. https://socket.io/docs/v4/client-offline-behavior/#buffered-events
- Client to server at least once: `io({ retries: 3, ackTimeout: 10000 })`. For exactly once, store a client-generated ID under a unique constraint. Acknowledge on success and on a duplicate, but do not acknowledge on any other error, so the client times out and retries. https://socket.io/docs/v4/tutorial/step-8
- Connection state recovery is opt-in, needs a bounded `maxDisconnectionDuration` and can fail: branch on `socket.recovered`. The Redis (Pub/Sub) adapter does not support it, the Redis Streams adapter does. https://socket.io/docs/v4/connection-state-recovery

```js
const io = new Server(httpServer, {
  connectionStateRecovery: { maxDisconnectionDuration: 2 * 60 * 1000, skipMiddlewares: false },
});
```

https://socket.io/docs/v4/connection-state-recovery#usage

### CORS and deployment

- CORS is off by default. The option is `cors.origin`, never `origins`. `origin: "*"` with `credentials: true` fails in the browser: list explicit origins. https://socket.io/docs/v4/server-options/#cors
- CORS covers only browsers on HTTP long-polling, not WebSocket. Restrict who connects with `allowRequest`. https://socket.io/docs/v4/handling-cors/
- A proxy must forward the `Upgrade` and `Connection` headers, and its read timeout must exceed `pingInterval + pingTimeout` (45 seconds by default), or clients see `transport close`. https://socket.io/docs/v4/reverse-proxy/#nginx
- With several servers and HTTP long-polling enabled, sticky sessions are still required whatever the adapter (the alternatives are disabling long-polling or client-side load balancing). https://socket.io/docs/v4/using-multiple-nodes/ and https://socket.io/docs/v4/adapter/
- Without sticky sessions, clients get HTTP 400 responses with `{"code":1,"message":"Session ID unknown"}`. https://socket.io/docs/v4/troubleshooting-connection-issues/#you-didnt-enable-sticky-sessions-in-a-multi-server-setup
- `path` (default `/socket.io/`) must match on both sides. `io("/my-path/")` selects a namespace, not a path. https://socket.io/docs/v4/troubleshooting-connection-issues/#the-request-path-does-not-match-on-both-sides
- Close with `io.close()`, which also closes the HTTP server. https://socket.io/docs/v4/server-api/#serverclosecallback

## Reference files

| File | Read it for |
| ---- | ----------- |
| [references/server-setup.md](references/server-setup.md) | Installation, init with Express, Koa, Fastify, Hono, HTTPS, HTTP/2, Bun, uWebSockets.js, server options and defaults, bundling, handler layout |
| [references/middleware-and-auth.md](references/middleware-and-auth.md) | Connection, socket and engine middlewares, credentials, user identity, header hooks, `allowRequest`, client certificates |
| [references/cors-and-proxies.md](references/cors-and-proxies.md) | CORS options and errors, cookies, nginx, Apache, Caddy, `http-proxy`, cloud load balancers, sharing a port with `ws` |
| [references/server-api.md](references/server-api.md) | Server and Socket instances, namespaces on `io`, utility methods, `serverSideEmit`, disconnect reasons, engine events and error codes, closing |
| [references/client.md](references/client.md) | Bundles, init, Socket events, `connect_error` and `disconnect`, Manager and multiplexing, every client option, bundlers |
| [references/events-rooms-namespaces.md](references/events-rooms-namespaces.md) | Serialization, acknowledgements, catch-all listeners, validation, error handling, broadcasting, rooms, dynamic namespaces |
| [references/delivery-and-recovery.md](references/delivery-and-recovery.md) | Delivery guarantees, retries, exactly once, offset replay, connection state recovery |
| [references/multiple-nodes.md](references/multiple-nodes.md) | Sticky sessions, proxy configs for several nodes, alternatives, Node.js cluster, PM2 |
| [references/adapters.md](references/adapters.md) | Redis, sharded Redis, Redis Streams, Postgres, MongoDB, Cluster, AWS SQS, Azure, GCP adapters, feature matrix, emitters |
| [references/performance-and-operations.md](references/performance-and-operations.md) | Native add-ons, `wsEngine`, memory, custom parsers, OS limits, load testing, Admin UI, debug logs |
| [references/typescript.md](references/typescript.md) | Event interfaces on server, client and namespaces, `SocketData`, typed tests |
| [references/testing-and-troubleshooting.md](references/testing-and-troubleshooting.md) | Server integration tests, client tests with a fake socket, connection failures, disconnections, stuck long-polling |
| [references/migrations-and-protocol.md](references/migrations-and-protocol.md) | v2 to v3 and v3 to v4 changes, rolling upgrade with `allowEIO3`, Engine.IO and Socket.IO wire protocols |

## Old patterns

| Old usage | Replacement | Source |
| --------- | ----------- | ------ |
| `origins` or `handlePreflightRequest` option | `cors: { origin: [...] }` | https://socket.io/docs/v4/migrating-from-2-x-to-3-0/#cors-handling |
| `io.set("authorization", fn)` | `io.use((socket, next) => ...)` | https://socket.io/docs/v4/migrating-from-2-x-to-3-0/#ioset-is-removed |
| Token in the Socket `query` option | `auth` option, read as `socket.handshake.auth` | https://socket.io/docs/v4/migrating-from-2-x-to-3-0/ |
| Client `error` event for a refused connection | `connect_error`, with `err.message` and `err.data` | https://socket.io/docs/v4/migrating-from-2-x-to-3-0/ |
| `socket.on("reconnect_attempt")` on the Socket | `socket.io.on("reconnect_attempt")` on the Manager | https://socket.io/docs/v4/client-socket-instance/#events |
| Client `pong` event for latency | `socket.volatile.emit("ping", cb)` on an interval | https://socket.io/docs/v4/migrating-from-2-x-to-3-0/#no-more-pong-event-for-retrieving-latency |
| `Namespace.connected` object | `Namespace.sockets`, a `Map` | https://socket.io/docs/v4/migrating-from-2-x-to-3-0/ |
| `Object.keys(socket.rooms)` | `socket.rooms` is a `Set` | https://socket.io/docs/v4/migrating-from-2-x-to-3-0/ |
| `socket.join(room, cb)` | `socket.join(room)`, awaited with an async adapter | https://socket.io/docs/v4/migrating-from-2-x-to-3-0/ |
| `Namespace.clients(cb)` (v2), then `allSockets()` (v3, to be removed in the next major) | `fetchSockets()` | https://socket.io/docs/v4/server-api/#namespaceallsockets |
| `socket.binary(false)` | A custom `parser` | https://socket.io/docs/v4/migrating-from-2-x-to-3-0/ |
| `socket.emit("a").emit("b")` | Separate calls, `emit()` returns `true` | https://socket.io/docs/v4/migrating-from-2-x-to-3-0/ |
| Default import of `socket.io` or `socket.io-client` | `import { Server } from "socket.io"`, `import { io } from "socket.io-client"` | https://socket.io/docs/v4/migrating-from-2-x-to-3-0/ |
| `@types/socket.io`, `@types/socket.io-client` | Bundled types | https://socket.io/docs/v4/migrating-from-2-x-to-3-0/ |
| `cookieName`, `cookieHttpOnly`, `cookiePath` | `cookie: { name, httpOnly, path }` | https://socket.io/docs/v4/migrating-from-2-x-to-3-0/#no-more-cookie-by-default |
| `socket.to(room).broadcast.emit()` | `socket.to(room).emit()` | https://socket.io/docs/v4/migrating-from-3-x-to-4-0/#known-migration-issues |
| `io.to(a)` in a loop, then `io.emit()` | `const op = io.to(a).to(b); op.emit()` | https://socket.io/docs/v4/migrating-from-3-x-to-4-0/#ioto-is-now-immutable |
| `wsEngine: "eiows"` string | `wsEngine: require("eiows").Server` | https://socket.io/docs/v4/migrating-from-3-x-to-4-0/#wsengine-option |
| `uws` | `eiows` | https://socket.io/docs/v4/server-installation/#other-websocket-server-implementations |
| `socket.io-redis` | `@socket.io/redis-adapter`, with your own Redis clients | https://socket.io/docs/v4/redis-adapter/#migrating-from-socketio-redis |
| `socket.io-emitter` | `@socket.io/redis-emitter`, with your own Redis client | https://socket.io/docs/v4/redis-adapter/#migrating-from-socketio-emitter |
| Express middleware wrapped for `io.use` | `io.engine.use(middleware)` | https://socket.io/docs/v4/middlewares/#compatibility-with-express-middleware |
| Artillery default Socket.IO engine (v2 client) | `artillery-engine-socketio-v3` | https://socket.io/docs/v4/load-testing/#artillery |
| v2 clients on a v4 server | `allowEIO3: true` during the upgrade, then back to `false` | https://socket.io/docs/v4/migrating-from-2-x-to-3-0/ |
