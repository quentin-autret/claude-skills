# CORS and reverse proxies (Socket.IO 4.8)

## Contents

- Enabling CORS
- Credentials and cookies
- What CORS does not protect
- CORS errors
- nginx
- Apache HTTPD
- Caddy 2, Node.js http-proxy, cloud load balancers
- Sharing the HTTP server with a plain WebSocket server

## Enabling CORS

- Cross-origin access is off by default. Enable it with the `cors` option, forwarded to the `cors` package. https://socket.io/docs/v4/handling-cors/#configuration

```js
const io = new Server(httpServer, {
  cors: { origin: ["https://example.com"] },
});
```

https://socket.io/docs/v4/handling-cors/#configuration

- The key is `origin`, even for several domains, never `origins`. https://socket.io/docs/v4/server-options/#cors
- `origin` takes a Boolean, String, RegExp, Array of strings and RegExps, or Function. `true` reflects the request origin, `false` disables CORS. https://socket.io/docs/v4/handling-cors/#configuration
- `origin: "*"` disables the protection of CORS. https://socket.io/docs/v4/server-options/#cors
- A development page on another port of `localhost` is another origin: allow it explicitly. https://socket.io/docs/v4/handling-cors/#configuration

```js
cors: {
  origin: process.env.NODE_ENV === "production" ? false : ["http://localhost:3000"],
}
```

https://socket.io/docs/v4/server-options/#cors

- `allowedHeaders` reflects `Access-Control-Request-Headers` by default, `exposedHeaders` is empty, `credentials` is omitted unless `true`. `optionsSuccessStatus` serves legacy browsers that reject 204. https://socket.io/docs/v4/handling-cors/#configuration

## Credentials and cookies

- Cookies need both sides: server `cors: { origin: [...], credentials: true }`, client `withCredentials: true`. https://socket.io/docs/v4/handling-cors/#configuration

```js
// client
const socket = io("https://my-backend.com", { withCredentials: true });
// server
const io = new Server(httpServer, {
  cors: { origin: "https://my-frontend.com", credentials: true },
});
```

https://socket.io/docs/v4/client-options/#withcredentials

- `origin: "*"` with `credentials: true` fails in the browser. List the domains. A function that always returns `true` with credentials removes CORS security. https://socket.io/docs/v4/handling-cors/#credential-is-not-supported-if-the-cors-header-access-control-allow-origin-is-
- The Node.js client sends cookies with `withCredentials: true`, which helps cookie-based sticky sessions. https://socket.io/docs/v4/client-options/#withcredentials

## What CORS does not protect

- CORS only applies to browsers. Scripts on a server and native apps ignore it. https://socket.io/docs/v4/handling-cors/
- CORS only applies to HTTP long-polling: WebSocket connections are not subject to it. https://socket.io/docs/v4/handling-cors/
- To actually restrict who connects, use `allowRequest`. This example rejects every cross-origin request: https://socket.io/docs/v4/handling-cors/#configuration

```js
const io = new Server(httpServer, {
  allowRequest: (req, callback) => {
    const noOriginHeader = req.headers.origin === undefined;
    callback(null, noOriginHeader);
  },
});
```

https://socket.io/docs/v4/handling-cors/#configuration

## CORS errors

- "Access-Control-Allow-Origin missing" with a correct config often means the server is unreachable. Test it: https://socket.io/docs/v4/handling-cors/#cors-header-access-control-allow-origin-missing

```sh
curl "https://api.example.com/socket.io/?EIO=4&transport=polling"
# expected: 0{"sid":"...","upgrades":["websocket"],"pingInterval":25000,"pingTimeout":20000,"maxPayload":1000000}
```

- "expected 'true' in Access-Control-Allow-Credentials": the client sets `withCredentials: true` but the server `cors` lacks `credentials: true`. https://socket.io/docs/v4/handling-cors/#expected-true-in-cors-header-access-control-allow-credentials

## nginx

- Forward the upgrade headers over HTTP/1.1. https://socket.io/docs/v4/reverse-proxy/#nginx

```nginx
location / {
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header Host $host;
  proxy_pass http://localhost:3000;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
}
```

https://socket.io/docs/v4/reverse-proxy/#nginx

- `proxy_read_timeout` (60 seconds by default) must exceed `pingInterval + pingTimeout` (45 seconds by default). Otherwise nginx closes idle connections and clients see `transport close`. https://socket.io/docs/v4/reverse-proxy/#nginx
- To proxy only Socket.IO traffic, use `location /socket.io/`. A custom `path` must be the same on server and client. https://socket.io/docs/v4/reverse-proxy/#nginx
- A proxy that drops the `Connection: upgrade` header leaves clients on long-polling and logs `connection_error` code 3, context `TRANSPORT_MISMATCH`. https://socket.io/docs/v4/troubleshooting-connection-issues/#a-proxy-in-front-of-your-servers-does-not-accept-the-websocket-connection

## Apache HTTPD

- Needs `mod_proxy`, `mod_proxy_http`, `mod_proxy_wstunnel` and `mod_rewrite`. `ProxyTimeout` must exceed `pingInterval + pingTimeout`. https://socket.io/docs/v4/reverse-proxy/#apache-httpd

```apache
ProxyPass / http://localhost:3000/
RewriteEngine on
RewriteCond %{HTTP:Upgrade} websocket [NC]
RewriteCond %{HTTP:Connection} upgrade [NC]
RewriteRule ^/?(.*) "ws://localhost:3000/$1" [P,L]
ProxyTimeout 60
```

https://socket.io/docs/v4/reverse-proxy/#apache-httpd

## Caddy 2, Node.js http-proxy, cloud load balancers

- Caddy 2: `reverse_proxy /socket.io/* localhost:3000`. For a custom path, `rewrite /path /path/`, then `handle_path /path/*` with `rewrite * /socket.io{path}` and `reverse_proxy localhost:3000`. https://socket.io/docs/v4/reverse-proxy/#caddy-2
- Node.js `http-proxy`: enable `ws: true`. https://socket.io/docs/v4/reverse-proxy/#nodejs-http-proxy

```js
const proxy = httpProxy.createProxyServer({ target: "http://localhost:3000", ws: true });
proxy.listen(80);
```

https://socket.io/docs/v4/reverse-proxy/#nodejs-http-proxy

- Google Cloud Application Load Balancer: the backend service timeout (30 seconds by default, except for serverless NEGs) also applies to WebSocket connections. The classic load balancer closes them after the timeout, idle or active. The global external and regional external load balancers close idle ones. Raise it above `pingInterval + pingTimeout` (45 seconds by default). https://socket.io/docs/v4/reverse-proxy/#google-cloud-application-load-balancer

```sh
gcloud compute backend-services update <BACKEND_SERVICE_NAME> --global --timeout=60
```

- Several nodes behind the proxy need sticky sessions, so that HTTP long-polling requests of a client reach the same server. They are not required with `transports: ["websocket"]`. https://socket.io/docs/v4/using-multiple-nodes/#sticky-sessions and https://socket.io/docs/v4/client-options/#transports

## Sharing the HTTP server with a plain WebSocket server

- Remove the default `upgrade` listeners and route by URL. https://socket.io/docs/v4/server-api/#enginehandleupgraderequest-socket-head

```js
httpServer.removeAllListeners("upgrade");
httpServer.on("upgrade", (req, socket, head) => {
  if (req.url === "/") {
    wss.handleUpgrade(req, socket, head, (ws) => wss.emit("connection", ws, req));
  } else if (req.url.startsWith("/socket.io/")) {
    io.engine.handleUpgrade(req, socket, head);
  } else {
    socket.destroy();
  }
});
```

https://socket.io/docs/v4/server-api/#enginehandleupgraderequest-socket-head
