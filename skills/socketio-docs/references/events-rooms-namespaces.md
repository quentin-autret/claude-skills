# Events, rooms and namespaces (Socket.IO 4.8)

## Contents

- Serialization
- Acknowledgements
- Volatile and compressed emits
- Catch-all listeners
- Validation and error handling
- Broadcasting
- Rooms
- Namespaces
- Dynamic namespaces

## Serialization

- Pass objects as they are, never `JSON.stringify()` them. https://socket.io/docs/v4/emitting-events/#basic-emit

```js
socket.emit("hello", { name: "John" });
```

- Any number of arguments and binary (`Buffer`, `TypedArray`, `ArrayBuffer`) are supported. A `Buffer` arrives as an `ArrayBuffer` in the browser, a `Uint8Array` arrives as a `Buffer` in Node.js. https://socket.io/docs/v4/tutorial/api-overview
- `Date` arrives as an ISO string. `Map` and `Set` must be converted: `[...myMap.entries()]`, `[...mySet.keys()]`. A class can define `toJSON()`. https://socket.io/docs/v4/emitting-events/#basic-emit

## Acknowledgements

- The ack callback is the last argument of `emit`, and the last argument of the handler. https://socket.io/docs/v4/emitting-events/#acknowledgements

```js
// server
socket.on("update item", (arg1, arg2, callback) => {
  callback({ status: "ok" });
});
// client
socket.emit("update item", "1", { name: "updated" }, (response) => {
  console.log(response.status);
});
```

https://socket.io/docs/v4/emitting-events/#acknowledgements

- `timeout(ms)` adds an error first argument: `(err, response)`. https://socket.io/docs/v4/emitting-events/#with-timeout
- The Promise form `emitWithAck()` is the alternative to the callback. Combined with `timeout()`, it rejects when no acknowledgement arrives in time. It works in both directions. https://socket.io/docs/v4/tutorial/api-overview

```js
try {
  const response = await socket.timeout(5000).emitWithAck("request", { foo: "bar" });
} catch (err) {
  // no acknowledgement in time
}
```

https://socket.io/docs/v4/tutorial/api-overview

- Broadcast acks need `timeout()`. The callback gets `(err, responses)`, 1 response per client, and works with `io.to(room)`, `socket.broadcast` and `io.of(ns)`. https://socket.io/docs/v4/broadcasting-events/#with-acknowledgements

```js
io.timeout(5000).emit("hello", "world", (err, responses) => {
  if (err) {
    // some clients did not acknowledge in time
  }
});
```

https://socket.io/docs/v4/broadcasting-events/#with-acknowledgements

- With a broadcast `emitWithAck`, the receiving side calls its callback with only 1 argument. https://socket.io/docs/v4/server-api/#serveremitwithackeventname-args

## Volatile and compressed emits

- `socket.volatile.emit(...)` drops the event when the connection is not ready. Use it when only the latest value matters, such as positions in a game. https://socket.io/docs/v4/emitting-events/#volatile-events
- `socket.compress(false).emit(...)` disables compression for that packet. https://socket.io/docs/v4/server-api/#socketcompressvalue

## Catch-all listeners

- `socket.onAny((eventName, ...args) => {})`, `prependAny`, `offAny`, and the outgoing set `onAnyOutgoing`, `prependAnyOutgoing`, `offAnyOutgoing`. https://socket.io/docs/v4/listening-to-events/#catch-all-listeners
- Acknowledgements bypass them: receiving an ack does not trigger `onAny`, sending one does not trigger `onAnyOutgoing`. https://socket.io/docs/v4/listening-to-events/#catch-all-listeners

## Validation and error handling

- Socket.IO does not validate arguments. Validate with a schema library, check that the callback is a function, and answer errors through it. https://socket.io/docs/v4/listening-to-events/#validation

```js
socket.on("create user", (payload, callback) => {
  if (typeof callback !== "function") {
    return socket.disconnect();
  }
  const { error, value } = userSchema.validate(payload);
  if (error) {
    return callback({ status: "Bad Request", error });
  }
  callback({ status: "OK" });
});
```

https://socket.io/docs/v4/listening-to-events/#validation

- Listeners have no error handling. Catch sync throws and rejected promises with a wrapper. https://socket.io/docs/v4/listening-to-events/#error-handling

```js
// handleError is your own logging or reporting function
const errorHandler = (handler) => (...args) => {
  try {
    const ret = handler.apply(this, args);
    if (ret && typeof ret.catch === "function") ret.catch(handleError);
  } catch (e) {
    handleError(e);
  }
};
socket.on("hello", errorHandler(() => { throw new Error("let's panic"); }));
```

https://socket.io/docs/v4/listening-to-events/#error-handling

## Broadcasting

- Broadcasting exists on the server only. `io.emit` reaches every connected client of the main namespace, `socket.broadcast.emit` everyone but the sender. https://socket.io/docs/v4/broadcasting-events/
- Clients that are disconnected at that moment never get the event. https://socket.io/docs/v4/broadcasting-events/#to-all-connected-clients
- `io.local.emit(...)` stays on the current server. Other servers are reached only through an adapter. https://socket.io/docs/v4/broadcasting-events/#with-multiple-socketio-servers
- Modifiers chain: this targets the union of `room1`, `room2`, `room3`, minus `room4`, minus the sender, with acks within 5 seconds. https://socket.io/docs/v4/emit-cheatsheet/

```js
socket.timeout(5000).to("room1").to(["room2", "room3"]).except("room4").emit("hello", (err, responses) => {});
```

https://socket.io/docs/v4/emit-cheatsheet/

## Rooms

- Rooms are server-side channels: a client cannot list its rooms. `to` and `in` are the same. https://socket.io/docs/v4/rooms/
- `socket.to(room)` excludes the sender, `io.to(room)` includes it. Every socket is in a room named after its ID: send a private message with `io.to(id).emit()`, since `socket.to(socket.id)` sends nothing. https://socket.io/docs/v4/server-api/#sockettoroom
- Name rooms after entities: `socket.join("project:" + project.id)`, or after the user ID to reach all of a user's tabs. https://socket.io/docs/v4/rooms/#sample-use-cases
- Rooms are left on disconnection. In `disconnecting` `socket.rooms` still holds them, in `disconnect` it is empty. https://socket.io/docs/v4/rooms/#disconnection
- The in-memory adapter keeps `rooms` and `sids` Maps at `io.of("/").adapter`. Never edit them, use `join` and `leave`. They are per server, not shared. https://socket.io/docs/v4/rooms/#implementation-details
- The adapter emits `create-room`, `delete-room` (room) and `join-room`, `leave-room` (room, id). https://socket.io/docs/v4/rooms/#room-events

```js
io.of("/").adapter.on("join-room", (room, id) => {
  console.log(`socket ${id} has joined room ${room}`);
});
```

https://socket.io/docs/v4/rooms/#room-events

## Namespaces

- A namespace has its own handlers, rooms and middlewares over a shared connection. `room1` in `/orders` is not `room1` in `/users`. https://socket.io/docs/v4/namespaces/#introduction
- `/` is the main namespace. `io.sockets` is `io.of("/")`. Connecting to `/admin` runs only the `/admin` middlewares and handlers. https://socket.io/docs/v4/namespaces/#main-namespace
- One connection carries several namespaces of the same origin, unless the namespace is created twice, the domains differ or `forceNew` is set. https://socket.io/docs/v4/namespaces/#client-initialization

## Dynamic namespaces

- Register by RegExp or function. The function receives the requested name, the client `auth` and `next(null, allowed)`. https://socket.io/docs/v4/namespaces/#dynamic-namespaces

```js
io.of((name, auth, next) => {
  next(null, isAllowed(name, auth));
}).on("connection", (socket) => {
  const namespace = socket.nsp;
});
```

https://socket.io/docs/v4/namespaces/#dynamic-namespaces

- A broad RegExp such as `/^\/\w+$/` lets any client create namespaces. When creation depends on identity or tenant, validate the name and `auth` in the function form. https://socket.io/docs/v4/namespaces/#security-considerations
- Explicitly registered namespaces take priority over dynamic ones. https://socket.io/docs/v4/namespaces/#priority
- The parent namespace returned by `io.of(regex|fn)` applies its middlewares to every child, and `parent.emit()` reaches all children. https://socket.io/docs/v4/namespaces/#parent-namespace
- Empty child namespaces stay in memory by default. Set `cleanupEmptyChildNamespaces: true` (since 4.6.0) to remove a child namespace, and close its adapter, once its last socket disconnects. https://socket.io/docs/v4/namespaces/#cleanup-of-empty-child-namespaces and https://socket.io/docs/v4/server-options/#cleanupemptychildnamespaces
