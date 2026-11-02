# node:tls and node:dtls

## Contents

- Defaults that surprise
- Ciphers and security level
- Trust store
- Session resumption
- PSK
- DTLS (added in v26.9.0)
- Old patterns

## Defaults that surprise

- `tls.connect()` sends no SNI by default, unlike `https`: set `servername` (a host name, never an IP) or the server may return the wrong certificate. https://nodejs.org/docs/latest/api/tls.html#tlsconnectoptions-callback
- Client `rejectUnauthorized` defaults to `true`. On a server it has effect only with `requestCert: true`, which defaults to `false`. https://nodejs.org/docs/latest/api/tls.html#tlscreateserveroptions-secureconnectionlistener
- Protocol range is TLSv1.2 to TLSv1.3. Choose it with `minVersion`/`maxVersion`. https://nodejs.org/docs/latest/api/tls.html#tlsdefault_min_version
- The client destroys connections whose DH parameter is below `minDHSize` (1024). https://nodejs.org/docs/latest/api/tls.html#tlsconnectoptions-callback
- Server `sessionTimeout` defaults to 300 s. https://nodejs.org/docs/latest/api/tls.html#tlscreateserveroptions-secureconnectionlistener
- ECDHE is on by default, DHE is off. Enable DHE with `dhparam: 'auto'` rather than custom parameters. https://nodejs.org/docs/latest/api/tls.html#perfect-forward-secrecy
- Resumed connections do not re-verify the peer: `authorized` carries the stored result. Since v26.8.0 a resumed TLS 1.3 session without a client certificate is no longer reported as authorized. https://nodejs.org/docs/latest/api/tls.html#tlssocketauthorized

## Ciphers and security level

- Keep the default cipher list unless absolutely necessary. Change it per context with `ciphers`, not globally. https://nodejs.org/docs/latest/api/tls.html#modifying-the-default-tls-cipher-suite
- Lower the OpenSSL security level (default 2) for legacy peers in the cipher string of that context only, not with `--tls-cipher-list`: https://nodejs.org/docs/latest/api/tls.html#setting-security-levels

```js
tls.createServer({ key, cert, ciphers: 'DEFAULT@SECLEVEL=0', minVersion: 'TLSv1' }, onSocket);
```

- Editing `tls.DEFAULT_CIPHERS` affects only sockets opened afterwards. https://nodejs.org/docs/latest/api/tls.html#modifying-the-default-tls-cipher-suite
- 3 of the 5 TLSv1.3 suites are on (the CCM ones are off). TLSv1.3 suites are toggled only by full `TLS_...` names. https://nodejs.org/docs/latest/api/tls.html#modifying-the-default-tls-cipher-suite
- Names from `tls.getCiphers()` are lowercase and must be uppercased when used. https://nodejs.org/docs/latest/api/tls.html#tlsgetciphers

## Trust store

- `tls.rootCertificates` is a fixed snapshot of the bundled Mozilla store, not what is actually used. Read the effective list with `tls.getCACertificates('default')` (also `'system'`, `'bundled'`, `'extra'`). https://nodejs.org/docs/latest/api/tls.html#tlsrootcertificates , https://nodejs.org/docs/latest/api/tls.html#tlsgetcacertificatestype
- `tls.setDefaultCACertificates()` (added v24.5.0) replaces the list for the current thread. Append to keep the defaults, and call it before any cached TLS connection: https://nodejs.org/docs/latest/api/tls.html#tlssetdefaultcacertificatescerts

```js
tls.setDefaultCACertificates([...tls.getCACertificates('default'), ...extra]);
```

- `allowPartialTrustChain` treats intermediate CAs in `ca` as anchors. https://nodejs.org/docs/latest/api/tls.html#tlscreatesecurecontextoptions
- `tlsSocket.setKeyCert(context)` picks a certificate from an `ALPNCallback`. https://nodejs.org/docs/latest/api/tls.html#tlssocketsetkeycertcontext
- `certificateCompression` (RFC 8879) and `tls.getCertificateCompressionAlgorithms()` are added in v26.4.0. https://nodejs.org/docs/latest/api/tls.html#tlsgetcertificatecompressionalgorithms

## Session resumption

- A failed resumption falls back silently to a full handshake. Check with `openssl s_client -connect localhost:443 -reconnect` (later lines say "Reused"). https://nodejs.org/docs/latest/api/tls.html#session-resumption
- Client: save the `'session'` event value (several may arrive with TLS 1.3) and pass it as `session` to the next `tls.connect()`. `getSession()` works only up to TLS 1.2. https://nodejs.org/docs/latest/api/tls.html#event-session
- Across workers or machines, share session IDs through `'newSession'`/`'resumeSession'`, or give every server the same 48-byte `ticketKeys` and rotate with `setTicketKeys()`. Ticket keys decrypt every TLS 1.2 session that used them: never store them on disk. https://nodejs.org/docs/latest/api/tls.html#session-identifiers , https://nodejs.org/docs/latest/api/tls.html#session-tickets
- Client renegotiation is limited to 3 per 600 s. Do not change it. https://nodejs.org/docs/latest/api/tls.html#client-initiated-renegotiation-attack-mitigation

## PSK

- TLS-PSK is off by default and has a history of OpenSSL flaws. Use high-entropy keys, never passwords. `pskCallback` cannot be async, and the client needs a custom `checkServerIdentity`. Return random data for an unknown identity to avoid revealing it. https://nodejs.org/docs/latest/api/tls.html#pre-shared-keys
- `enableTrace()` output is unstable, do not parse it. https://nodejs.org/docs/latest/api/tls.html#tlssocketenabletrace

## DTLS (added in v26.9.0)

Active development (Stability 1.1). Needs `--experimental-dtls` at build time and at run time, plus `--allow-net` under the Permission Model. https://nodejs.org/docs/latest/api/dtls.html

- `dtls.listen(cb, { cert, key, port })` returns an endpoint serving many sessions on one UDP socket. `dtls.connect(host, port, opts)` returns a session (`send`, `onmessage`, `await session.opened`). https://nodejs.org/docs/latest/api/dtls.html#dtlslistencallback-options
- Defaults: `rejectUnauthorized: true` (on `listen` only with `requestCert`, default `false`), `mtu` 1200, `handshakeTimeout` 60000, `maxSessions` 10000, `maxSessionsPerHost` 1000 (raise it behind a shared NAT). Refused peers get silence by design. https://nodejs.org/docs/latest/api/dtls.html#denial-of-service
- A wrong PSK or no common cipher gives no error until `handshakeTimeout`: invalid records are discarded, not answered. https://nodejs.org/docs/latest/api/dtls.html#handshake-timeout
- A PSK function must return synchronously. `psk` without `ciphers` enables PSK suites, and CoAP's `PSK-AES128-CCM8` needs `@SECLEVEL=0`. https://nodejs.org/docs/latest/api/dtls.html#pre-shared-keys
- DTLS `ticketKeys` are 80 bytes, not 48. The default key is random per context, so tickets die on restart. https://nodejs.org/docs/latest/api/dtls.html#ticket-keys

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `secureProtocol: 'TLSv1_2_method'` | `minVersion` / `maxVersion` | https://nodejs.org/docs/latest/api/tls.html#tlscreatesecurecontextoptions |
| `clientCertEngine`, `privateKeyEngine`, `privateKeyIdentifier` | OpenSSL 3 providers | https://nodejs.org/docs/latest/api/deprecations.html#DEP0183 |
| `tls.rootCertificates` as the effective CA list | `tls.getCACertificates()` | https://nodejs.org/docs/latest/api/tls.html#tlsrootcertificates |
| `require('node:_tls_wrap')`, `node:_tls_common` | `node:tls` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0192 |
| `--tls-cipher-list` to allow a legacy peer | per-context `ciphers` | https://nodejs.org/docs/latest/api/tls.html#setting-security-levels |
