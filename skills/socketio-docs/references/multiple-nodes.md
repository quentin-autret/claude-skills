# Running several nodes (Socket.IO 4.8)

## Contents

- Two separate jobs
- Sticky sessions
- nginx
- nginx Ingress on Kubernetes
- Apache HTTPD, HAProxy, Traefik
- Alternatives to sticky sessions
- Node.js cluster
- PM2

## Two separate jobs

- Scaling needs both load balancing (every request of a session reaches the process that created it) and an adapter (broadcasts reach clients on other servers). https://socket.io/docs/v4/using-multiple-nodes/
- Sticky sessions are required with any adapter as long as HTTP long-polling is enabled. Without them, clients get HTTP 400 `{"code":1,"message":"Session ID unknown"}`. https://socket.io/docs/v4/troubleshooting-connection-issues/#you-didnt-enable-sticky-sessions-in-a-multi-server-setup
- Without an adapter, `io.emit()` and room broadcasts reach only clients of the current server. https://socket.io/docs/v4/adapter/

## Sticky sessions

- Route by cookie, client IP, or a session identifier in the URL or a header. https://socket.io/docs/v4/using-multiple-nodes/#sticky-sessions
- The proxy read timeout must exceed `pingInterval + pingTimeout` (45 seconds by default), or clients see `transport close`. https://socket.io/docs/v4/using-multiple-nodes/#nginx-configuration
- A cookie-based balancer can reuse the `io` cookie, which the server sends only when the `cookie` option is set. https://socket.io/docs/v4/server-options/#cookie

```js
const io = new Server(httpServer, {
  cookie: { name: "io", httpOnly: true, sameSite: "strict" },
});
```

https://socket.io/docs/v4/server-options/#cookie

- A cross-origin browser client sends that cookie only with `withCredentials: true` and a server CORS config with `credentials: true`. https://socket.io/docs/v4/client-options/#withcredentials

## nginx

- `hash $remote_addr consistent;` hashes the full client IP. `ip_hash` uses only the first 3 octets of IPv4. The `sticky` cookie directive needs the commercial subscription. https://socket.io/docs/v4/using-multiple-nodes/#nginx-configuration

```nginx
http {
  server {
    listen 3000;
    location / {
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $host;
      proxy_pass http://nodes;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
    }
  }
  upstream nodes {
    hash $remote_addr consistent;
    server app01:3000;
    server app02:3000;
  }
}
```

https://socket.io/docs/v4/using-multiple-nodes/#nginx-configuration

## nginx Ingress on Kubernetes

- Behind another proxy, every request carries the proxy IP. Hash on the first `X-Forwarded-For` address instead: extract it into `$client_ip` and set `nginx.ingress.kubernetes.io/upstream-hash-by: "$client_ip"`. https://socket.io/docs/v4/using-multiple-nodes/#nginx-ingress-kubernetes
- Kubernetes cookie affinity, AWS ALB sticky sessions, GCP session affinity and Heroku session affinity are the platform options. https://socket.io/docs/v4/using-multiple-nodes/#sticky-sessions

## Apache HTTPD, HAProxy, Traefik

- Apache: 2 balancers, `balancer://nodes_polling` and `balancer://nodes_ws`, each with `stickysession=SERVERID`, and a rewrite on `Upgrade: websocket` that sends WebSocket requests to the second. `ProxyTimeout 60`. https://socket.io/docs/v4/using-multiple-nodes/#apache-httpd-configuration
- HAProxy: reuse the `io` cookie with `cookie io prefix indirect nocache` and an HTTP check on `/health`. https://socket.io/docs/v4/using-multiple-nodes/#haproxy-configuration
- Traefik: a sticky cookie through `loadBalancer.sticky.cookie.name`, with `httpOnly=true`. https://socket.io/docs/v4/using-multiple-nodes/#traefik

## Alternatives to sticky sessions

- Disable long-polling on the client. A single WebSocket connection needs no stickiness, but clients on networks that block WebSocket cannot connect. https://socket.io/docs/v4/using-multiple-nodes/#disabling-http-long-polling

```js
const socket = io({ transports: ["websocket"] });
```

https://socket.io/docs/v4/using-multiple-nodes/#disabling-http-long-polling

- Client-side balancing: change `socket.io.uri` on each reconnection attempt. Topology changes need a config or discovery endpoint, and random choice spreads load unevenly. https://socket.io/docs/v4/using-multiple-nodes/#client-side-load-balancing

```js
socket.io.on("reconnect_attempt", () => {
  socket.io.uri = pickServer();
});
```

https://socket.io/docs/v4/using-multiple-nodes/#client-side-load-balancing

- Syncing session state between servers to avoid stickiness is not offered, because of the performance cost. https://socket.io/docs/v4/using-multiple-nodes/

## Node.js cluster

- Default to `@socket.io/sticky` (routes on the `sid` query parameter) with `@socket.io/cluster-adapter`. The HTTP server listens in the primary only. https://socket.io/docs/v4/cluster-adapter/#with-nodejs-cluster

```js
import cluster from "node:cluster";
import http from "node:http";
import { availableParallelism } from "node:os";
import { Server } from "socket.io";
import { setupMaster, setupWorker } from "@socket.io/sticky";
import { createAdapter, setupPrimary } from "@socket.io/cluster-adapter";

if (cluster.isPrimary) {
  const httpServer = http.createServer();
  setupMaster(httpServer, { loadBalancingMethod: "least-connection" });
  setupPrimary();
  cluster.setupPrimary({ serialization: "advanced" }); // needed for Buffer payloads
  httpServer.listen(3000);
  for (let i = 0; i < availableParallelism(); i++) cluster.fork();
  cluster.on("exit", () => cluster.fork());
} else {
  const httpServer = http.createServer();
  const io = new Server(httpServer);
  io.adapter(createAdapter());
  setupWorker(io);
}
```

https://socket.io/docs/v4/cluster-adapter/#with-nodejs-cluster

- `serialization: "advanced"` is only needed when packets carry Buffers. https://socket.io/docs/v4/cluster-adapter/#with-nodejs-cluster
- Other sticky packages route on other keys: `sticky-session` on `connection.remoteAddress`, `socketio-sticky-session` on `x-forwarded-for`. https://socket.io/docs/v4/using-multiple-nodes/#using-nodejs-cluster
- A cluster covers 1 host. Across hosts, use a networked adapter such as Redis. https://socket.io/docs/v4/pm2/#how-it-works

## PM2

- Stock `pm2` does not route a session to one worker. Default to `@socket.io/pm2`, a drop-in fork: remove `pm2` first. https://socket.io/docs/v4/pm2/#installation

```sh
npm remove -g pm2
npm install -g @socket.io/pm2
```

- In the worker, set the cluster adapter, call `setupWorker(io)`, and never call `httpServer.listen()`: the PM2 God process owns the HTTP server. https://socket.io/docs/v4/pm2/#usage

```js
const { createServer } = require("http");
const { Server } = require("socket.io");
const { createAdapter } = require("@socket.io/cluster-adapter");
const { setupWorker } = require("@socket.io/sticky");

const httpServer = createServer();
const io = new Server(httpServer);
io.adapter(createAdapter());
setupWorker(io);

process.on("SIGINT", () => {
  io.close((err) => process.exit(err ? 1 : 0));
});
```

https://socket.io/docs/v4/pm2/#usage

- Run it with `instances: "max"` and `exec_mode: "cluster"` in the PM2 config. https://socket.io/docs/v4/pm2/#usage
- Other options with PM2: a distinct port per worker behind nginx, or WebSocket-only clients. https://socket.io/docs/v4/pm2/
