# Delivery guarantees and connection state recovery (Socket.IO 4.8)

## Contents

- What is guaranteed by default
- Client to server: at least once
- Exactly once with a unique ID
- Server to client: replay from an offset
- Connection state recovery
- Recovery and adapters

## What is guaranteed by default

- Ordering is guaranteed on every transport, upgrade included, for messages that arrive. https://socket.io/docs/v4/delivery-guarantees#message-ordering
- Arrival is at most once. A message in flight when the connection breaks can be lost, with no retry. The server has no buffer, so a disconnected client misses events for good. https://socket.io/docs/v4/delivery-guarantees#at-most-once
- Socket.IO stores no message. Persisting messages for offline clients is the application's job. https://socket.io/docs/v4/faq/#does-socketio-store-the-messages
- Messages are also lost when the server crashes or restarts while processing, or when the database is down. https://socket.io/docs/v4/tutorial/step-8
- The client buffers emits while offline, then flushes them on reconnection. https://socket.io/docs/v4/tutorial/step-8
- Disconnections are normal (proxy restarts, autoscaling, WiFi to 4G, frozen tabs). Design for resynchronization after each one. https://socket.io/docs/v4/troubleshooting-connection-issues/#problem-the-socket-gets-disconnected

## Client to server: at least once

- Default to the `retries` option with `ackTimeout`. Events are queued and sent one by one, so order is kept. https://socket.io/docs/v4/tutorial/step-8

```js
const socket = io({ ackTimeout: 10000, retries: 3 });
```

https://socket.io/docs/v4/tutorial/step-8

- The server must acknowledge each event, or the client resends it up to `retries + 1` times. https://socket.io/docs/v4/client-options/#retries
- Pending events are still lost when the user refreshes the tab. https://socket.io/docs/v4/delivery-guarantees#from-client-to-server
- A manual retry on ack timeout also works but does not keep order. https://socket.io/docs/v4/tutorial/step-8

```js
function emit(socket, event, arg) {
  socket.timeout(5000).emit(event, arg, (err) => {
    if (err) emit(socket, event, arg);
  });
}
```

https://socket.io/docs/v4/tutorial/step-8

## Exactly once with a unique ID

- Retries can deliver the same event twice. Attach a client-generated ID and store it in a UNIQUE column. https://socket.io/docs/v4/tutorial/step-8
- On a duplicate (constraint error), still acknowledge with `callback()`. On any other error, do not acknowledge, so the client retries. https://socket.io/docs/v4/tutorial/step-8

```js
// client
let counter = 0;
const clientOffset = `${socket.id}-${counter++}`;
socket.emit("chat message", msg, clientOffset);

// server
socket.on("chat message", async (msg, clientOffset, callback) => {
  try {
    await db.run("INSERT INTO messages (content, client_offset) VALUES (?, ?)", msg, clientOffset);
  } catch (e) {
    if (isUniqueConstraintError(e)) {
      callback(); // already stored: acknowledge anyway
    }
    return; // other errors: no ack, the client retries
  }
  callback();
});
```

https://socket.io/docs/v4/tutorial/step-8

## Server to client: replay from an offset

- Give each event a unique ID, persist it, keep the last received ID on the client in `socket.auth`, and replay what is missing on connection. https://socket.io/docs/v4/delivery-guarantees#from-server-to-client

```js
// client
const socket = io({ auth: { offset: undefined } });
socket.on("my-event", ({ id, data }) => {
  socket.auth.offset = id;
});

// server
io.on("connection", async (socket) => {
  const offset = socket.handshake.auth.offset;
  if (offset) {
    for (const event of await fetchMissedEventsFromDatabase(offset)) {
      socket.emit("my-event", event);
    }
  }
});
```

https://socket.io/docs/v4/delivery-guarantees#from-server-to-client

- Combine it with recovery: replay only when `socket.recovered` is `false`. https://socket.io/docs/v4/tutorial/step-7

```js
io.on("connection", async (socket) => {
  if (!socket.recovered) {
    // replay rows with id > socket.handshake.auth.serverOffset
  }
});
```

https://socket.io/docs/v4/tutorial/step-7

## Connection state recovery

- Opt-in on the server. After a temporary disconnection it restores `id`, rooms, `data` and the missed packets. https://socket.io/docs/v4/connection-state-recovery

```js
const io = new Server(httpServer, {
  connectionStateRecovery: {
    maxDisconnectionDuration: 2 * 60 * 1000,
    skipMiddlewares: false,
  },
});

io.on("connection", (socket) => {
  if (socket.recovered) {
    // id, rooms and data were restored
  } else {
    // new session or unrecoverable: resynchronize
  }
});
```

https://socket.io/docs/v4/connection-state-recovery#usage

- Recovery does not always succeed (a crash or restart loses the state). Always keep a resync path. https://socket.io/docs/v4/tutorial/step-6
- Keep `maxDisconnectionDuration` bounded, never `Infinity`: the feature targets short outages. https://socket.io/docs/v4/connection-state-recovery#usage
- State is saved only on an unexpected disconnection, not after `socket.disconnect()`. https://socket.io/docs/v4/connection-state-recovery#usage
- The server must have sent at least 1 event so the client holds an offset. The private session ID (`pid`) differs from the public `id`. https://socket.io/docs/v4/connection-state-recovery#how-it-works-under-the-hood
- `skipMiddlewares: true` lets a recovered client skip middlewares, including a user banned in the meantime. https://socket.io/docs/v4/connection-state-recovery#skipmiddlewares-option
- On the client, `socket.recovered` tells the same thing. https://socket.io/docs/v4/client-api/#socketrecovered
- To force a reconnection in a test, raise `reconnectionDelay` and `reconnectionDelayMax`, then call `socket.io.engine.close()`. https://socket.io/docs/v4/connection-state-recovery#usage

## Recovery and adapters

| Adapter | Recovery |
| ------- | -------- |
| In-memory (default) | Yes |
| Redis (Pub/Sub) | No |
| Redis Streams | Yes |
| MongoDB | Yes, adapter 0.3.0 and later |
| Postgres, Cluster | No |
| AWS SQS, Azure Service Bus, GCP Pub/Sub | No |

Sources: https://socket.io/docs/v4/connection-state-recovery#compatibility-with-existing-adapters, https://socket.io/docs/v4/redis-streams-adapter/#supported-features, https://socket.io/docs/v4/postgres-adapter/#supported-features, https://socket.io/docs/v4/cluster-adapter/#supported-features (the recovery page itself lists Postgres and Cluster as work in progress)

- To scale with recovery, use the Redis Streams adapter. With recovery on, it stores sessions in Redis as key/value pairs. https://socket.io/docs/v4/redis-streams-adapter/#how-it-works
