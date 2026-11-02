# Middleware and authentication (Socket.IO 4.8)

## Contents

- Connection middleware
- Sending credentials
- Identifying a user
- Socket middleware (per packet)
- Engine middleware (Express)
- Header hooks and allowRequest
- Client certificates

## Connection middleware

- `io.use((socket, next) => ...)` runs once per connection, even when the connection spans several HTTP requests. Uses: logging, authentication, authorization, rate limiting. https://socket.io/docs/v4/middlewares/
- Middlewares run in order. After `next(err)`, the next ones do not run. https://socket.io/docs/v4/middlewares/#registering-a-middleware
- Call `next()` in every code path. A middleware that never calls it leaves the connection hanging until it is closed after a timeout (`connectTimeout`, 45000 ms by default, closes a client that has not joined a namespace). https://socket.io/docs/v4/server-options/#connecttimeout
- The socket is not connected while middlewares run: no `disconnect` event fires if it fails, and `connection` does not fire either. https://socket.io/docs/v4/middlewares/#registering-a-middleware
- `next(err)` refuses the connection. The client gets `connect_error` with `err.message` and `err.data`. https://socket.io/docs/v4/middlewares/#handling-middleware-error

```js
io.use((socket, next) => {
  const err = new Error("not authorized");
  err.data = { content: "Please retry later" };
  next(err);
});
```

https://socket.io/docs/v4/middlewares/#handling-middleware-error

- Middlewares are per namespace: `io.use` guards `/` only. Use `io.of("/admin").use(...)` for another namespace, or attach to every namespace, dynamic ones included: https://socket.io/docs/v4/server-api/#event-new_namespace

```js
io.on("new_namespace", (namespace) => namespace.use(myMiddleware));
```

https://socket.io/docs/v4/server-api/#event-new_namespace

- A middleware on a parent namespace (returned by `io.of(regex)` or `io.of(fn)`) applies to each child namespace. https://socket.io/docs/v4/namespaces/#parent-namespace
- With connection state recovery and `skipMiddlewares: true`, a recovered client skips middlewares, so a user blocked during the disconnection can come back. https://socket.io/docs/v4/connection-state-recovery#skipmiddlewares-option

## Sending credentials

- The client sends credentials in `auth`, an object or a function that receives a callback. The server reads `socket.handshake.auth`. https://socket.io/docs/v4/middlewares/#sending-credentials

```js
// client
const socket = io({ auth: (cb) => { cb({ token: localStorage.token }); } });
// server
io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  // validate, then next() or next(err)
});
```

https://socket.io/docs/v4/client-options/#auth

- `query` parameters appear in `socket.handshake.query` and cannot be updated during a session: a change only applies once the current session closes and a new one starts. https://socket.io/docs/v4/client-options/#query
- After a refusal by a middleware, the client does not reconnect automatically (`socket.active` is false): update `socket.auth` and call `socket.connect()`. https://socket.io/docs/v4/client-socket-instance/
- To force a reconnection with new credentials: `socket.auth.token = "efgh"; socket.disconnect().connect();`. https://socket.io/docs/v4/client-options/#auth
- `socket.handshake.headers` and `socket.request.headers` hold the headers of the first HTTP request of the session only. https://socket.io/docs/v4/server-api/#sockethandshake
- Browsers ignore `extraHeaders` when only the WebSocket transport is enabled, because the browser WebSocket API cannot set headers. https://socket.io/docs/v4/client-options/#extraheaders

## Identifying a user

- Socket.IO has no user concept. Reuse a Passport or `express-session` context, or validate `auth` credentials in a middleware. https://socket.io/docs/v4/faq/#how-to-identify-a-given-user
- `socket.id` is an ephemeral identifier, not meant for application use unless connection state recovery is enabled. https://socket.io/docs/v4/server-socket-instance/#socketid
- Attach your own attribute, such as `socket.user`, as long as it does not overwrite an existing one. https://socket.io/docs/v4/server-socket-instance/#additional-attributes
- Store data meant to be read across servers in `socket.data`. https://socket.io/docs/v4/server-socket-instance/#socketdata

## Socket middleware (per packet)

- `socket.use(([event, ...args], next) => ...)` runs for each incoming packet. `next(err)` stops the packet and emits `error` on the server socket. https://socket.io/docs/v4/server-socket-instance/#socket-middlewares

```js
io.on("connection", (socket) => {
  socket.use(([event, ...args], next) => {
    if (isUnauthorized(event)) return next(new Error("unauthorized event"));
    next();
  });
  socket.on("error", (err) => {
    if (err && err.message === "unauthorized event") socket.disconnect();
  });
});
```

https://socket.io/docs/v4/server-socket-instance/#socket-middlewares

- A socket middleware that calls neither `next()` nor `next(err)` drops the packet silently: no handler, no error, no acknowledgement. For an authorization failure, call `next(new Error(...))`. https://socket.io/docs/v4/server-socket-instance/#socket-middlewares
- Socket middlewares exist on the server only. The client equivalent is a catch-all listener. https://socket.io/docs/v4/server-socket-instance/#socket-middlewares

## Engine middleware (Express)

- Socket.IO middlewares are not Express middlewares. Register Express middlewares with `io.engine.use(...)`, which runs for every HTTP request, upgrade requests included. https://socket.io/docs/v4/middlewares/#compatibility-with-express-middleware

```js
io.engine.use(helmet());
io.engine.use(session({
  secret: "keyboard cat",
  resave: false,
  saveUninitialized: true,
  cookie: { secure: true },
}));
```

https://socket.io/docs/v4/middlewares/#compatibility-with-express-middleware

- Run an engine middleware on the handshake only by testing that the `sid` query parameter is absent. https://socket.io/docs/v4/middlewares/#compatibility-with-express-middleware

```js
io.engine.use((req, res, next) => {
  const isHandshake = req._query.sid === undefined;
  if (isHandshake) {
    passport.authenticate("jwt", { session: false })(req, res, next);
  } else {
    next();
  }
});
```

https://socket.io/docs/v4/middlewares/#compatibility-with-express-middleware

## Header hooks and allowRequest

- `io.engine.on("initial_headers", (headers, req) => ...)` runs for the handshake response only, `headers` for every HTTP response including the WebSocket upgrade. Set a cookie once in `initial_headers`. https://socket.io/docs/v4/server-api/#event-initial_headers

```js
io.engine.on("initial_headers", (headers, request) => {
  headers["set-cookie"] = serialize("uid", "1234", { sameSite: "strict" });
});
```

https://socket.io/docs/v4/server-api/#event-initial_headers

- Header hooks are synchronous. Do async work, such as a session lookup, in `allowRequest` and store the result on `req`. https://socket.io/docs/v4/server-api/#event-initial_headers

```js
const io = new Server(httpServer, {
  allowRequest: async (req, callback) => {
    req.session = await fetchSession(req);
    callback(null, true);
  },
});
```

https://socket.io/docs/v4/server-api/#event-initial_headers

- A request refused by `allowRequest` shows as `connection_error` code 4, "Forbidden". https://socket.io/docs/v4/troubleshooting-connection-issues/#troubleshooting-steps

## Client certificates

- Create the HTTPS server with `requestCert: true` and `ca`. The peer certificate is only readable during the engine `connection` event, so copy it there. https://socket.io/docs/v4/server-initialization/#with-an-https-server

```js
io.engine.on("connection", (rawSocket) => {
  rawSocket.peerCertificate = rawSocket.request.client.getPeerCertificate();
});
io.on("connection", (socket) => {
  console.log(socket.conn.peerCertificate);
});
```

https://socket.io/docs/v4/server-initialization/#with-an-https-server

- A Node.js client sends its certificate with the `cert` and `key` options, and trusts a self-signed server with `ca`. https://socket.io/docs/v4/client-options/#nodejs-specific-options
