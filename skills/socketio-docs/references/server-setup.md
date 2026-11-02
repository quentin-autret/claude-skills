# Server setup (Socket.IO 4.8)

## Contents

- Installation
- Initialization by framework
- Runtimes other than Node.js HTTP
- Server options and defaults
- Bundling the server
- Handler layout

## Installation

- Make sure Node.js is installed (at least Node.js 10). The current LTS release is an ideal starting point. https://socket.io/docs/v4/server-installation/
- Install the optional `ws` native add-ons. They improve certain operations. Without them the WebSocket server falls back to the JavaScript implementation. https://socket.io/docs/v4/server-installation/#additional-packages

```sh
npm install --save-optional bufferutil utf-8-validate
```

- Type declarations of dependencies ship inside the package. https://socket.io/docs/v4/server-installation/#dependency-tree
- `socket.io` 4.8.x depends on `engine.io` 6.6.x, which depends on `ws` 8.17.x. https://socket.io/docs/v4/server-installation/#transitive-versions

## Initialization by framework

- Standalone: `new Server(3000, options)`, or `new Server(options)` then `io.listen(3000)`. This starts an HTTP server reachable as `io.httpServer`. https://socket.io/docs/v4/server-initialization/#standalone
- Existing HTTP server: `new Server(httpServer, options)`, then `httpServer.listen(port)`. https://socket.io/docs/v4/server-initialization/#with-an-http-server
- Express: wrap `app` with `createServer(app)` and call `listen` on that server, never `app.listen()`. https://socket.io/docs/v4/server-initialization/#with-express
- Koa: `new Server(createServer(app.callback()))`. https://socket.io/docs/v4/server-initialization/#with-koa
- Fastify: pass `fastify.server` and close the sockets in a `preClose` hook. https://socket.io/docs/v4/server-initialization/#with-fastify

```js
const io = new Server(fastify.server, { /* options */ });
fastify.addHook("preClose", (done) => {
  io.local.disconnectSockets(true);
  done();
});
fastify.listen({ port: 3000 });
```

https://socket.io/docs/v4/server-initialization/#with-fastify

- Hono on Node.js: pass the value returned by `serve(...)` of `@hono/node-server` (cast to `HTTPServer` in TypeScript). https://socket.io/docs/v4/server-initialization/#with-hono-nodejs
- HTTPS: `https.createServer({ key, cert })`. https://socket.io/docs/v4/server-initialization/#with-an-https-server
- HTTP/2: `http2.createSecureServer({ allowHTTP1: true, key, cert })`. https://socket.io/docs/v4/server-initialization/#with-an-http2-server
- NestJS 8 and later embed Socket.IO v4, NestJS 7 and earlier embed Socket.IO v2: match the client version. https://socket.io/docs/v4/server-initialization/#with-nest

## Runtimes other than Node.js HTTP

- uWebSockets.js: `io.attachApp(app)`. https://socket.io/docs/v4/server-installation/#usage-with-uwebsockets

```js
const { App } = require("uWebSockets.js");
const { Server } = require("socket.io");
const app = App();
const io = new Server();
io.attachApp(app);
app.listen(3000, (token) => { if (!token) console.warn("port already in use"); });
```

https://socket.io/docs/v4/server-installation/#usage-with-uwebsockets

- Bun: use `@socket.io/bun-engine` with `io.bind(engine)`. Bun's `idleTimeout` (seconds) must exceed `pingInterval` (25 seconds by default). Adapters work unchanged. https://socket.io/docs/v4/server-installation/#usage-with-bun

```js
const io = new Server();
const engine = new Engine({ path: "/socket.io/" });
io.bind(engine);
export default { port: 3000, idleTimeout: 30, ...engine.handler() };
```

https://socket.io/docs/v4/server-installation/#usage-with-bun

- Hono on Bun uses the same engine and routes `/socket.io/` to `engine.handleRequest(req, server)`. https://socket.io/docs/v4/server-initialization/#with-hono--bun
- Most serverless platforms bill by handler duration, so long-lived Socket.IO connections are not recommended there. https://socket.io/docs/v4/troubleshooting-connection-issues/#deployment-on-a-serverless-platform

## Server options and defaults

| Option | Default | Note |
| ------ | ------- | ---- |
| `path` | `/socket.io/` | Must match the client unless a proxy rewrites it |
| `serveClient` | `true` | Serves `/socket.io/socket.io.js` and other bundles. Set `false` to not serve the client files |
| `addTrailingSlash` | `true` | `false` accepts `/socket.io` |
| `adapter` | in-memory | Replace it to scale |
| `cors` | none | Options forwarded to the `cors` module |
| `connectionStateRecovery` | disabled | Opt-in |
| `cleanupEmptyChildNamespaces` | `false` | Removes empty dynamic namespaces and their adapter |
| `connectTimeout` | `45000` | Ms before disconnecting a client that joined no namespace |
| `pingInterval` | `25000` | Server sends a ping every interval |
| `pingTimeout` | `20000` | Ms to receive the pong |
| `maxHttpBufferSize` | `1e6` | Bytes per message before the socket is closed |
| `perMessageDeflate` | `false` | Costly in CPU and memory |
| `httpCompression` | `true` | Long-polling compression, threshold 1024 |
| `transports` | `["polling", "websocket"]` | Add `"webtransport"` to enable it |
| `allowUpgrades` | `true` | `upgradeTimeout` default `10000` |
| `cookie` | none | Options forwarded to the `cookie` module |
| `allowEIO3` | `false` | Accepts v2 clients |
| `allowRequest` | none | `(req, callback)` decides on each handshake or upgrade |
| `parser` | `socket.io-parser` | Same parser on both sides |
| `wsEngine` | `require("ws").Server` | Alternative WebSocket server class, such as `require("eiows").Server` |

Sources: https://socket.io/docs/v4/server-options/

- `pingInterval` and `pingTimeout`: a very small `pingInterval` (such as 1000) loads the server noticeably with a few thousand clients. A small `pingTimeout` causes reconnections when the server is briefly busy, a large one delays detection. React Native warns above 60 seconds for their sum. https://socket.io/docs/v4/server-options/#pingtimeout
- A paused debugger breaks the heartbeat: raise both values while debugging. https://socket.io/docs/v4/server-options/#pingtimeout
- `perMessageDeflate: false` makes `socket.compress(true)` a no-op on WebSocket, `httpCompression: false` makes it a no-op on long-polling. https://socket.io/docs/v4/server-options/#permessagedeflate
- No cookie is sent by default since v3. The cookie enables sticky sessions, which are required with several servers and HTTP long-polling, but not needed for a single server or IP-based sticky sessions. https://socket.io/docs/v4/migrating-from-2-x-to-3-0/#no-more-cookie-by-default
- WebTransport needs HTTPS, which is why it is not on by default. https://socket.io/docs/v4/faq/#what-is-webtransport
- A custom `io.engine.generateId = (req) => uuid.v4()` must be unique across all servers. https://socket.io/docs/v4/server-instance/#serverengine

## Bundling the server

- Webpack 5: `target: "node"`, `serveClient: false`. The optional add-ons `bufferutil` and `utf-8-validate` can also be set as externals. https://socket.io/docs/v4/server-with-bundlers/#without-serving-the-client-files

```js
externals: { bufferutil: "bufferutil", "utf-8-validate": "utf-8-validate" }
```

- To serve the client from a bundle, load the files as `asset/source` modules and override `Server.sendFile`. https://socket.io/docs/v4/server-with-bundlers/#including-serving-the-client-files

```js
Server.sendFile = (filename, req, res) => {
  res.end(filename.endsWith(".map") ? clientMap : clientFile);
};
```

## Handler layout

- One suggested layout: each module exports `registerXHandlers(io, socket)`, called from 1 `onConnection` handler. https://socket.io/docs/v4/server-application-structure/

```js
const onConnection = (socket) => {
  registerOrderHandlers(io, socket);
  registerUserHandlers(io, socket);
};
io.on("connection", onConnection);
```

https://socket.io/docs/v4/server-application-structure/

- When handlers are factories registered in `index.js`, write them as `function` expressions and use `this` as the socket: arrow functions lose it. https://socket.io/docs/v4/server-application-structure/#all-event-handlers-are-registered-in-the-indexjs-file
- The examples name events by domain, such as `"order:create"` and `"user:update-password"`. https://socket.io/docs/v4/server-application-structure/
