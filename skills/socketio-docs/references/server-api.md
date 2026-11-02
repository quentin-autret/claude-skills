# Server API (Socket.IO 4.8)

## Contents

- The Server targets the main namespace
- Engine access and counts
- Utility methods
- serverSideEmit
- Server-side Socket properties
- Disconnection events and reasons
- Engine events and error codes
- Closing and memory

## The Server targets the main namespace

- `io.on`, `io.use`, `io.emit`, `io.to`, `io.in`, `io.except`, `io.timeout`, `io.fetchSockets`, `io.socketsJoin`, `io.socketsLeave`, `io.disconnectSockets`, `io.serverSideEmit`, `io.local`, `io.volatile` and `io.sockets` all act on `/`. `new_namespace` is emitted by the Server itself. https://socket.io/docs/v4/server-api/#methods
- The Server emits one event, `connection` (`connect` is an alias). https://socket.io/docs/v4/server-instance/#events
- Constructors: `new Server(httpServer, opts)`, `new Server(port, opts)`, or `new Server(opts)` then `io.listen(port)` or `io.attach(server)`. https://socket.io/docs/v4/server-api/#constructor
- `io.serveClient(false)` has no effect once `listen` has run. https://socket.io/docs/v4/server-api/#serverserveclientvalue
- `server.bind(engine)` and `server.onconnection(socket)` are for advanced use only. https://socket.io/docs/v4/server-api/#serverbindengine

## Engine access and counts

- `io.engine` is the Engine.IO server. `io.engine.clientsCount` counts low-level connections and can differ from `io.of("/").sockets.size`. https://socket.io/docs/v4/server-instance/#serverengine
- `namespace.sockets.size` counts sockets on the current node only. https://socket.io/docs/v4/server-api/#namespacesockets
- `io.engine.generateId = (req) => uuid.v4()` replaces the session ID generator (base64id by default). IDs must be unique across all servers. https://socket.io/docs/v4/server-api/#enginegenerateid

## Utility methods

- `socketsJoin`, `socketsLeave`, `disconnectSockets` and `fetchSockets` take the broadcast filters `of`, `in`/`to`, `except` and `local`. With a compatible adapter they act on the whole cluster. `local` limits them to the current node. https://socket.io/docs/v4/server-instance/#utility-methods

```js
io.of("/admin").in("room1").except("room2").local.disconnectSockets();
const sockets = await io.in(theSocketId).fetchSockets();
```

https://socket.io/docs/v4/migrating-from-3-x-to-4-0/

- `disconnectSockets(true)` also closes the low-level connection, `disconnectSockets()` leaves only the namespace. Same meaning as `socket.disconnect(close)`. https://socket.io/docs/v4/server-api/#socketdisconnectclose
- `fetchSockets()` returns `Socket` or `RemoteSocket` objects with `id`, `handshake`, `rooms`, `data`, `emit`, `join`, `leave` and `disconnect` only. https://socket.io/docs/v4/server-api/#namespacefetchsockets
- `socket.data` is the object shared across servers through `fetchSockets()`. https://socket.io/docs/v4/server-api/#socketdata

```js
io.on("connection", (socket) => { socket.data.username = "alice"; });
const sockets = await io.fetchSockets();
console.log(sockets[0].data.username);
```

https://socket.io/docs/v4/server-api/#socketdata

- Detect the last connection of a user: join a room named after the user ID, and on `disconnect` check `(await io.in(userId).fetchSockets()).length === 0`. https://socket.io/docs/v4/server-api/#serverfetchsockets
- `allSockets()` is to be removed in the next major version. Use `fetchSockets()` or `serverSideEmit()`. https://socket.io/docs/v4/server-api/#namespaceallsockets

## serverSideEmit

- `io.serverSideEmit(event, ...args, ack)` reaches the other Socket.IO servers of the cluster, not the sender. Arguments are `JSON.stringify`-ed, so binary is not supported. https://socket.io/docs/v4/server-api/#serversideemiteventname-args-ack

```js
io.serverSideEmit("ping", (err, responses) => {
  // err: at least 1 server did not respond in time, responses holds the others
});
io.on("ping", (cb) => cb("pong")); // on the other servers
```

https://socket.io/docs/v4/server-instance/#serversideemit

- `connection`, `connect` and `new_namespace` are reserved names for `serverSideEmit`. https://socket.io/docs/v4/server-instance/#serversideemit
- `serverSideEmitWithAck` is the Promise form. https://socket.io/docs/v4/server-api/#serversideemiteventname-args-ack

## Server-side Socket properties

- `socket.id`: 20 random characters, synced with the client, read-only, regenerated on every reconnection unless the state is recovered. https://socket.io/docs/v4/server-socket-instance/#socketid
- `socket.handshake`: `headers`, `query`, `auth`, `time`, `issued`, `url`, `address`, `xdomain`, `secure`. `headers` and `query` come from the first HTTP request and never update. https://socket.io/docs/v4/server-socket-instance/#sockethandshake
- `socket.rooms`: a `Set` that starts with the socket ID. https://socket.io/docs/v4/server-socket-instance/#socketrooms
- `socket.conn`: the Engine.IO socket. `socket.conn.transport.name` is `polling`, then `websocket` after the upgrade. Events: `upgrade`, `packet`, `packetCreate`, `drain`, `close`. https://socket.io/docs/v4/server-socket-instance/#socketconn
- `socket.recovered`: `true` when connection state recovery restored `id`, `rooms` and `data`. https://socket.io/docs/v4/server-api/#socketrecovered
- `socket.nsp`: the namespace of the socket. https://socket.io/docs/v4/namespaces/#custom-namespaces
- `socket.join()` and `socket.leave()` can return a Promise with some adapters: await them there. https://socket.io/docs/v4/server-api/#socketjoinroom
- `socket.compress(false)` disables compression for the next emit. https://socket.io/docs/v4/server-api/#socketcompressvalue

## Disconnection events and reasons

- `disconnecting` fires before `disconnect`, while `socket.rooms` is still filled: notify rooms there. In an async handler, copy the set before the first `await`. https://socket.io/docs/v4/server-api/#event-disconnecting

```js
socket.on("disconnecting", (reason) => {
  for (const room of socket.rooms) {
    if (room !== socket.id) socket.to(room).emit("user has left", socket.id);
  }
});
```

https://socket.io/docs/v4/server-socket-instance/#disconnecting

```js
socket.on("disconnecting", async (reason) => {
  const rooms = new Set(socket.rooms);
  await someLongRunningOperation();
  // use rooms here: socket.rooms is empty after the first await
});
```

https://socket.io/docs/v4/server-api/#event-disconnecting

- Reasons passed to `disconnect`: `server namespace disconnect`, `client namespace disconnect` (the client called `socket.disconnect()`), `server shutting down`, `ping timeout` (no pong within `pingTimeout`), `transport close`, `transport error`, `parse error`, `forced close`, `forced server close` (no namespace joined within `connectTimeout`). https://socket.io/docs/v4/server-api/#event-disconnect

## Engine events and error codes

- `io.engine.on("connection_error", (err) => ...)` exposes `err.req`, `err.code`, `err.message`, `err.context`. https://socket.io/docs/v4/server-instance/#serverengine

| Code | Message | Usual cause |
| ---- | ------- | ----------- |
| 0 | Transport unknown | Should not happen |
| 1 | Session ID unknown | Sticky sessions missing |
| 2 | Bad handshake method | Should not happen |
| 3 | Bad request | Proxy not forwarding WebSocket headers |
| 4 | Forbidden | Refused by `allowRequest` |
| 5 | Unsupported protocol version | Client and server versions incompatible |

Source: https://socket.io/docs/v4/troubleshooting-connection-issues/#troubleshooting-steps

- `initial_headers` fires for the handshake response, `headers` for every response including the upgrade. https://socket.io/docs/v4/server-instance/#serverengine

## Closing and memory

- `io.close([callback])` disconnects every client and closes the HTTP server. Closing only the HTTP server leaves WebSocket clients connected. https://socket.io/docs/v4/server-api/#serverclosecallback
- Each session keeps a reference to its first HTTP request, needed by `express-session`. Drop it when unused: https://socket.io/docs/v4/memory-usage/

```js
io.engine.on("connection", (rawSocket) => {
  rawSocket.request = null;
});
```

https://socket.io/docs/v4/performance-tuning/#discard-the-initial-http-request
