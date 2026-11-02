# node:net, node:dns and node:dgram

## Contents

- TCP sockets
- TCP servers
- IPC paths
- BlockList and handle transfer
- DNS
- UDP
- Old patterns

## TCP sockets

- Attach `'error'` to every socket and server. A socket `'error'` is followed by `'close'`, a server `'error'` is not unless you call `close()`. https://nodejs.org/docs/latest/api/net.html#event-error_1 , https://nodejs.org/docs/latest/api/net.html#event-error
- `socket.setTimeout()` only emits `'timeout'`: call `end()` or `destroy()` yourself. There is no timeout by default. https://nodejs.org/docs/latest/api/net.html#socketsettimeouttimeout-callback

```js
socket.setTimeout(3000);
socket.on('timeout', () => socket.end());
```

- `write()` returns `false` when data is queued in user memory, so memory can grow: wait for `'drain'`. https://nodejs.org/docs/latest/api/net.html#socketwritedata-encoding-callback
- `end()` half-closes (FIN), `destroy()` closes now, `resetAndDestroy()` sends RST. `allowHalfOpen` defaults to `false`. https://nodejs.org/docs/latest/api/net.html#socketendata-encoding-callback , https://nodejs.org/docs/latest/api/net.html#netcreateserveroptions-connectionlistener
- Nagle is on for plain `net` sockets (`noDelay: false`), off for `http.Server`. `setNoDelay()` with no argument disables it. https://nodejs.org/docs/latest/api/net.html#socketsetnodelaynodelay , https://nodejs.org/docs/latest/api/http.html#httpcreateserveroptions-requestlistener
- `setKeepAlive()` rounds milliseconds down to whole seconds (400 ms becomes 0). `interval` and `count` parameters and the options-object form are added in v26.4.0. https://nodejs.org/docs/latest/api/net.html#socketsetkeepalive
- `autoSelectFamily` (Happy Eyeballs) is on by default since v20. All failures arrive as one `AggregateError`. It is ignored when `family` is set or `localAddress` is given. The docs disagree on the attempt timeout (250 vs 500 ms): set it if it matters. https://nodejs.org/docs/latest/api/net.html#socketconnectoptions-connectlistener
- `typeOfService` / `setTypeOfService()` are added in v25.6.0. https://nodejs.org/docs/latest/api/net.html#socketsettypeofservicetos

## TCP servers

- Handle `EADDRINUSE` in the server `'error'` event. `listen()` again only after an error or `close()`. https://nodejs.org/docs/latest/api/net.html#serverlisten

```js
server.on('error', (e) => {
  if (e.code === 'EADDRINUSE') setTimeout(() => { server.close(); server.listen(PORT, HOST); }, 1000);
});
```

- Read an OS-assigned port from `server.address()` after `'listening'`. It is `null` before. https://nodejs.org/docs/latest/api/net.html#serveraddress
- Backlog defaults to 511. Without `host` the server binds `::` (usually dual-stack). `ipv6Only: true` disables dual stack. https://nodejs.org/docs/latest/api/net.html#serverlistenport-host-backlog-callback
- `maxConnections = 0` drops every connection since v21 (it was unlimited). https://nodejs.org/docs/latest/api/net.html#servermaxconnections
- `reusePort: true` load-balances one port across sockets on Linux 3.9+, FreeBSD 12+ and a few others, and errors elsewhere. https://nodejs.org/docs/latest/api/net.html#serverlistenoptions-callback
- `server.close()` waits for existing connections, and `'close'` fires once all end. https://nodejs.org/docs/latest/api/net.html#serverclosecallback
- `pauseOnConnect: true` lets you hand sockets to another process before reading. https://nodejs.org/docs/latest/api/net.html#netcreateserveroptions-connectionlistener

## IPC paths

- Unix socket paths are limited to about 107 bytes on Linux, 103 on macOS, and stale files persist after a crash: unlink them. A leading `\0` is a Linux abstract socket. Windows pipes must be under `\\?\pipe\` or `\\.\pipe\`. https://nodejs.org/docs/latest/api/net.html#ipc-support

```js
net.createServer().listen(path.join('\\\\?\\pipe', process.cwd(), 'myctl'));
```

- A socket created as root may be unreachable to others: use `readableAll` / `writableAll`. https://nodejs.org/docs/latest/api/net.html#ipc-support

## BlockList and handle transfer

- Filter IPs with `net.BlockList` on `createServer({ blockList })` or `new Socket({ blockList })`. Behind a proxy or NAT the inbound check sees the proxy address. https://nodejs.org/docs/latest/api/net.html#netcreateserveroptions-connectionlistener
- `BlockList.PRIVATE_RANGES`, `addCIDRs()`, `remove*()` and `size` are added in v26.8.0: https://nodejs.org/docs/latest/api/net.html#blocklistprivate_ranges

```js
const blockList = new net.BlockList();
blockList.addCIDRs(net.BlockList.PRIVATE_RANGES);
```

- `net.BoundSocket` (added in v26.4.0) reserves a port synchronously, adopted later by `server.listen(bound)` or `new net.Socket({ handle: bound })`. Close a never-adopted handle. https://nodejs.org/docs/latest/api/net.html#class-netboundsocket
- Transfer a fresh, not-yet-reading TCP socket or a listening server to a worker with `postMessage(value, [socket])`. Otherwise `ERR_WORKER_HANDLE_NOT_TRANSFERABLE`. https://nodejs.org/docs/latest/api/net.html#transferring-tcp-handles-to-other-threads

## DNS

- `dns.lookup()` is not a DNS query: it calls `getaddrinfo` (honors `/etc/hosts`) synchronously on the libuv threadpool, so slow lookups starve fs, crypto and zlib. `dns.resolve*()` query the network, ignore `/etc/hosts` and do not use the threadpool. https://nodejs.org/docs/latest/api/dns.html#implementation-considerations , https://nodejs.org/docs/latest/api/dns.html#dnslookup
- Many networking APIs call `dns.lookup()` internally. `socket.connect()` and `dgram.createSocket()` accept a custom `lookup` option. https://nodejs.org/docs/latest/api/dns.html#dnslookup
- `ENOTFOUND` from `lookup` can also mean no free file descriptors. https://nodejs.org/docs/latest/api/dns.html#dnslookuphostname-options-callback
- Result order defaults to `'verbatim'`. Use `order` (`'ipv4first'`, `'ipv6first'`), not the deprecated boolean `verbatim`. `dns.setDefaultResultOrder()` in the main thread does not affect workers. https://nodejs.org/docs/latest/api/dns.html#dnssetdefaultresultorderorder
- `dns.setServers()` affects only `resolve*`/`reverse`. A `NOTFOUND` from the first server does not fall through to the next one. Never call it while a query runs. https://nodejs.org/docs/latest/api/dns.html#dnssetserversservers
- A `new dns.Resolver({ timeout, tries })` has its own servers. `resolver.cancel()` aborts its queries with `ECANCELLED`. https://nodejs.org/docs/latest/api/dns.html#class-dnsresolver
- Prefer specific `resolve4`/`resolveMx`... over `resolveAny()`, which servers may refuse. https://nodejs.org/docs/latest/api/dns.html#dnsresolveanyhostname-callback

## UDP

- Datagrams above 65,507 bytes are impossible, and those above the path MTU are dropped silently. https://nodejs.org/docs/latest/api/dgram.html#note-about-udp-datagram-size
- On a connected socket do not pass port/address to `send()`. Offsets count bytes. The send callback is the only signal that the buffer can be reused. A host name delays the send by a DNS lookup. https://nodejs.org/docs/latest/api/dgram.html#socketsendmsg-offset-length-port-address-callback
- `bindSync()` / `connectSync()` (added in v26.4.0) throw synchronously and need a numeric IP. https://nodejs.org/docs/latest/api/dgram.html#socketbindsyncoptions
- `reusePort` distributes datagrams across sockets, while with `reuseAddr` only one socket receives. https://nodejs.org/docs/latest/api/dgram.html#dgramcreatesocketoptions-callback
- When a UDP socket is shared across `cluster` workers, call `addMembership()` only once, or `EADDRINUSE` follows. https://nodejs.org/docs/latest/api/dgram.html#socketaddmembershipmulticastaddress-multicastinterface

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `socket.bufferSize` | `socket.writableLength` | https://nodejs.org/docs/latest/api/net.html#socketbuffersize |
| `server.listen({ signal })` as cleanup | `await using` or `close()` | https://nodejs.org/docs/latest/api/net.html#serverlistenoptions-callback |
| `--enable-network-family-autoselection` | `--network-family-autoselection` | https://nodejs.org/docs/latest/api/net.html#socketconnectoptions-connectlistener |
| `server._listen2()` | `server.listen()` | https://nodejs.org/docs/latest/api/deprecations.html#dep0208-serverprototype_listen2 |
| `dns.lookup(h, { verbatim: true })` | `{ order: 'verbatim' }` | https://nodejs.org/docs/latest/api/dns.html#dnssetdefaultresultorderorder |
