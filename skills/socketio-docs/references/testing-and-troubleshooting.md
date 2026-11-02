# Testing and troubleshooting (Socket.IO 4.8)

## Contents

- Server integration tests
- Client tests with a fake socket
- The client cannot connect
- Version compatibility
- Unexpected disconnections
- Stuck on HTTP long-polling
- Common mistakes

## Server integration tests

- Test the server with a real server and a real client. Listen without a port to get a free one, capture the server socket in `connection`, and start once the client emits `connect`. https://socket.io/docs/v4/testing/

```js
import { createServer } from "node:http";
import { Server } from "socket.io";
import { io as ioc } from "socket.io-client";

let io, serverSocket, clientSocket;

before((done) => {
  const httpServer = createServer();
  io = new Server(httpServer);
  io.on("connection", (socket) => { serverSocket = socket; });
  httpServer.listen(() => {
    const port = httpServer.address().port;
    clientSocket = ioc(`http://localhost:${port}`);
    clientSocket.on("connect", done);
  });
});

after(() => {
  io.close();
  clientSocket.disconnect();
});
```

https://socket.io/docs/v4/testing/

- Test acknowledgements with `await clientSocket.emitWithAck("foo")`. https://socket.io/docs/v4/testing/
- Wait for an event on either side with a small helper. https://socket.io/docs/v4/testing/

```js
function waitFor(socket, event) {
  return new Promise((resolve) => {
    socket.once(event, resolve);
  });
}

it("should work", async () => {
  serverSocket.emit("hello", "world");
  const arg = await waitFor(clientSocket, "hello");
  assert.equal(arg, "world");
});
```

https://socket.io/docs/v4/testing/

## Client tests with a fake socket

- Test client code in isolation, not against a server. Wrap `io(url)` in a module, inject the socket into services, and replace it with a fake in tests. https://socket.io/docs/v4/client-testing/

```js
export function createOrderService(socket) {
  return {
    createOrder(order) {
      socket.emit("order:create", order);
    },
    onOrderCreated(listener) {
      socket.on("order:created", listener);
      return () => {
        socket.off("order:created", listener);
      };
    },
  };
}
```

https://socket.io/docs/v4/client-testing/

- The fake implements only the methods used (`emit` as a mock, `on`, `off`) over a Map of listeners, plus `trigger(event, ...args)` to simulate a server event. https://socket.io/docs/v4/client-testing/
- Inject the service into components through props. In React, wrap `socket.trigger(...)` in `await act(async () => {...})` and return the unsubscribe function from `useEffect`. In Vue, unsubscribe in `onUnmounted` and `await wrapper.vm.$nextTick()` after a trigger. https://socket.io/docs/v4/client-testing/

## The client cannot connect

- Read `err.message`, `err.description` and `err.context` in the client `connect_error`, and `io.engine.on("connection_error", ...)` on the server. https://socket.io/docs/v4/troubleshooting-connection-issues/#problem-the-socket-is-not-able-to-connect

| Server code | Usual cause |
| ----------- | ----------- |
| 1 Session ID unknown | Sticky sessions missing on several servers |
| 3 Bad request | Proxy not forwarding WebSocket headers |
| 4 Forbidden | Refused by `allowRequest` |
| 5 Unsupported protocol version | Client and server majors incompatible |

Source: https://socket.io/docs/v4/troubleshooting-connection-issues/#troubleshooting-steps

- Check reachability. https://socket.io/docs/v4/troubleshooting-connection-issues/#the-server-is-not-reachable

```sh
curl "<server URL>/socket.io/?EIO=4&transport=polling"
# expected: 0{"sid":"...","upgrades":["websocket"],"pingInterval":25000,"pingTimeout":20000,"maxPayload":1000000}
```

- The client cannot reach a plain WebSocket server, even with `transports: ["websocket"]`. https://socket.io/docs/v4/troubleshooting-connection-issues/#you-are-trying-to-reach-a-plain-websocket-server
- "Cross-Origin Request Blocked": the server is unreachable or CORS is not enabled on it. https://socket.io/docs/v4/troubleshooting-connection-issues/#the-server-does-not-send-the-necessary-cors-headers
- The `path` differs between sides. `io("/my-custom-path/")` connects to a namespace of that name on `/socket.io/`. Set `path` on both. https://socket.io/docs/v4/troubleshooting-connection-issues/#the-request-path-does-not-match-on-both-sides

```js
new Server({ path: "/my-custom-path/" });
io(SERVER_URL, { path: "/my-custom-path/" });
```

## Version compatibility

- JS clients 3.x and 4.x work with servers 3.x and 4.x. A 2.x client needs `allowEIO3: true`, otherwise it gets HTTP 400 `{"code":5,"message":"Unsupported protocol version"}`. A 1.x client only works with a 1.x server. https://socket.io/docs/v4/troubleshooting-connection-issues/#the-client-is-not-compatible-with-the-version-of-the-server
- Java client 2.x works with 3.x and 4.x servers, 1.x needs `allowEIO3: true`. Swift 16.x needs `.version(.two)` for a 2.x server, Swift 15.x needs `allowEIO3: true` on a 4.x server. https://socket.io/docs/v4/troubleshooting-connection-issues/#the-client-is-not-compatible-with-the-version-of-the-server
- An incompatible v2 client may connect over WebSocket, then drop after about 30 seconds. https://socket.io/docs/v4/troubleshooting-connection-issues/#the-client-is-not-compatible-with-the-version-of-the-server-1

## Unexpected disconnections

- Disconnections at a regular interval point to a proxy timeout below `pingInterval + pingTimeout` (45 seconds by default). Raise nginx `proxy_read_timeout` or Apache `ProxyTimeout`. https://socket.io/docs/v4/troubleshooting-connection-issues/#something-between-the-server-and-the-client-closes-the-connection
- A message over `maxHttpBufferSize` (1 MB by default) closes the connection, and so does an upload longer than `pingTimeout`. Raise both. https://socket.io/docs/v4/troubleshooting-connection-issues/#you-are-trying-to-send-a-huge-payload

```js
const io = new Server(httpServer, { maxHttpBufferSize: 1e8, pingTimeout: 60000 });
```

https://socket.io/docs/v4/troubleshooting-connection-issues/#you-are-trying-to-send-a-huge-payload

- Read `(reason, details)` in the client `disconnect` handler. `details` carries `message`, `description` and `context`. https://socket.io/docs/v4/troubleshooting-connection-issues/#problem-the-socket-gets-disconnected

## Stuck on HTTP long-polling

- A WebSocket request without HTTP 101 means something blocks the upgrade. The connection still works, less efficiently. https://socket.io/docs/v4/troubleshooting-connection-issues/#problem-the-socket-is-stuck-in-http-long-polling
- Read the transport: client `socket.io.engine.transport.name`, server `socket.conn.transport.name`, with an `upgrade` event on each. https://socket.io/docs/v4/troubleshooting-connection-issues/#problem-the-socket-is-stuck-in-http-long-polling
- A proxy that drops `Connection: upgrade` gives `connection_error` code 3 with context `TRANSPORT_MISMATCH`. Check `err.req.headers`. https://socket.io/docs/v4/troubleshooting-connection-issues/#a-proxy-in-front-of-your-servers-does-not-accept-the-websocket-connection
- `express-status-monitor` runs its own Socket.IO instance and can interfere. https://socket.io/docs/v4/troubleshooting-connection-issues/#express-status-monitor-runs-its-own-socketio-instance

## Common mistakes

- Listeners registered inside the client `connect` handler run once per reconnection. Register them at top level. https://socket.io/docs/v4/troubleshooting-connection-issues/#duplicate-event-registration
- Server listeners registered after an `await` in `connection` miss early packets. Register first, await after. https://socket.io/docs/v4/troubleshooting-connection-issues/#delayed-event-handler-registration

```js
io.on("connection", async (socket) => {
  socket.on("hello", () => {});
  await longRunningOperation();
});
```

https://socket.io/docs/v4/troubleshooting-connection-issues/#delayed-event-handler-registration

- Messages sent to a `socket.id` after a reconnection are lost: the ID changed. Identify users with a session ID sent in a cookie or in `auth`. https://socket.io/docs/v4/troubleshooting-connection-issues/#usage-of-the-socketid-attribute
- A debugger pause breaks the heartbeat: raise `pingInterval` and `pingTimeout` while debugging. https://socket.io/docs/v4/server-options/#pingtimeout
