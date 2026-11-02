# TypeScript (Socket.IO 4.8)

## Contents

- Event interfaces
- Server
- Client
- Namespaces
- Tests
- Frameworks

## Event interfaces

- Declare 4 interfaces: events the server emits, events the client emits, events between servers, and the shape of `socket.data`. https://socket.io/docs/v4/typescript/#types-for-the-server

```ts
interface ServerToClientEvents {
  noArg: () => void;
  basicEmit: (a: number, b: string, c: Buffer) => void;
  withAck: (d: string, callback: (e: number) => void) => void;
}

interface ClientToServerEvents {
  hello: () => void;
}

interface InterServerEvents {
  ping: () => void;
}

interface SocketData {
  name: string;
  age: number;
}
```

https://socket.io/docs/v4/typescript/#types-for-the-server

## Server

- Generic order: `Server<ClientToServerEvents, ServerToClientEvents, InterServerEvents, SocketData>`. `ServerToClientEvents` types `emit` and broadcasts, `ClientToServerEvents` types `on`. https://socket.io/docs/v4/typescript/#types-for-the-server

```ts
const io = new Server<
  ClientToServerEvents,
  ServerToClientEvents,
  InterServerEvents,
  SocketData
>(httpServer);

io.on("connection", (socket) => {
  socket.emit("basicEmit", 1, "2", Buffer.from([3]));
  socket.data.name = "john";
});

io.serverSideEmit("ping");
```

https://socket.io/docs/v4/typescript/#types-for-the-server

- The Promise form infers the ack type: `const e = await socket.timeout(5_000).emitWithAck("withAck", "some string")`. https://socket.io/docs/v4/typescript/#types-for-the-server
- Types are hints, not validation: still validate and sanitize every payload. https://socket.io/docs/v4/typescript/#types-for-the-server
- Without type parameters, events are untyped and arguments are `any`. https://socket.io/docs/v4/migrating-from-3-x-to-4-0/

## Client

- The client reverses the first 2 parameters: `Socket<ServerToClientEvents, ClientToServerEvents>`. https://socket.io/docs/v4/typescript/#types-for-the-client

```ts
import { io, Socket } from "socket.io-client";

const socket: Socket<ServerToClientEvents, ClientToServerEvents> = io();

socket.on("withAck", (d, callback) => {
  callback(42);
});
socket.emit("hello");
```

https://socket.io/docs/v4/typescript/#types-for-the-client

- Do not install `@types/socket.io-client`: the package ships its own types, and the old ones cause errors such as "'extraHeaders' does not exist in type 'ConnectOpts'". https://socket.io/docs/v4/client-installation/#from-npm

## Namespaces

- Each namespace can have its own interfaces. https://socket.io/docs/v4/typescript/#custom-types-for-each-namespace

```ts
// server
const myNamespace: Namespace<NsClientToServer, NsServerToClient, NsInterServer, NsSocketData> =
  io.of("/my-namespace");

// client
const nsSocket: Socket<NsServerToClient, NsClientToServer> = io("/my-namespace");
```

https://socket.io/docs/v4/typescript/#custom-types-for-each-namespace

## Tests

- Import the client as `io as ioc`, and the socket types as `Socket as ClientSocket` from `socket.io-client` and `Socket as ServerSocket` from `socket.io`. Cast the bound address to `AddressInfo`. https://socket.io/docs/v4/testing/

```ts
import { createServer } from "node:http";
import { type AddressInfo } from "node:net";
import { io as ioc, type Socket as ClientSocket } from "socket.io-client";
import { Server, type Socket as ServerSocket } from "socket.io";

const httpServer = createServer();
const io = new Server(httpServer);
httpServer.listen(() => {
  const port = (httpServer.address() as AddressInfo).port;
  const clientSocket: ClientSocket = ioc(`http://localhost:${port}`);
});
```

https://socket.io/docs/v4/testing/

## Frameworks

- Hono on Node.js: cast the value of `serve(...)` to `HTTPServer` before passing it to `new Server`. https://socket.io/docs/v4/server-initialization/#with-hono-nodejs
- Postgres adapter: add `@types/pg`. https://socket.io/docs/v4/postgres-adapter/#installation
