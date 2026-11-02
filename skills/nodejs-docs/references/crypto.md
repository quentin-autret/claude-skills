# node:crypto: keys, randomness, hashing, KDFs

## Contents

- Keys
- Randomness and comparison
- Hashing and MACs
- Password KDFs
- Strings and algorithm strength
- Availability
- Version-flagged APIs
- Old patterns

## Keys

- Import key material into a `KeyObject` once and reuse it: repeated parsing is avoided and it is the fastest path. The first operation on a KeyObject can be slower (lazy OpenSSL cache). PEM strings and `{ key, format }` objects are for one-shot use. https://nodejs.org/docs/latest/api/crypto.html#key-formats
- JWK is the fastest serialized import for RSA. DER needs an explicit `type` (`'spki'`, `'pkcs8'`). https://nodejs.org/docs/latest/api/crypto.html#key-formats
- Store public keys as `'spki'` and private keys as encrypted `'pkcs8'` with a strong passphrase. https://nodejs.org/docs/latest/api/crypto.html#cryptogeneratekeypairtype-options-callback
- Raw formats `'raw-public'`, `'raw-private'`, `'raw-seed'` are in active development. `'raw-public'` is usually the fastest public import, the private ones are not always faster (the public key must be derived). https://nodejs.org/docs/latest/api/crypto.html#raw-key-formats
- Convert a Web Crypto `CryptoKey` to a `KeyObject` explicitly before calling `node:crypto` functions. Passing a `CryptoKey` is a runtime deprecation in v26.0.0 and will throw. https://nodejs.org/docs/latest/api/deprecations.html#DEP0203
- `KeyObject.from()` with a non-extractable `CryptoKey` is deprecated (v26.0.0) and will throw. https://nodejs.org/docs/latest/api/deprecations.html#DEP0204
- `x509.verify(publicKey)` checks only the signature, no other certificate validation. https://nodejs.org/docs/latest/api/crypto.html#x509verifypublickey
- Keys from OpenSSL STORE loaders (a `URL` as `key`) need `--allow-openssl-store` under the Permission Model and may reach files, devices, tokens and the network outside fs/net scopes. A `pkcs11:` URI does not prove which loader served the key. https://nodejs.org/docs/latest/api/crypto.html#private-keys-from-openssl-store-loaders

## Randomness and comparison

- Compare MACs, tokens and secrets with `crypto.timingSafeEqual(a, b)`. Both must have the same byte length or it throws, so check length first. Only the comparison is constant-time, surrounding code can still leak. On `Float32Array`/`Float64Array` it may give unexpected results. https://nodejs.org/docs/latest/api/crypto.html#cryptotimingsafeequala-b
- Use `crypto.randomInt(min, max)` for unbiased integers (`max` exclusive, range below 2^48). Never fill float arrays with `randomFill` for random floats. https://nodejs.org/docs/latest/api/crypto.html#cryptorandomfillbuffer-offset-size-callback
- `randomUUID()` caches entropy for up to 128 UUIDs. Pass `{ disableEntropyCache: true }` to bypass it. https://nodejs.org/docs/latest/api/crypto.html#cryptorandomuuidoptions
- `crypto.randomUUIDv7()` (added in v26.1.0) gives time-sortable keys, but its timestamp is not guaranteed strictly increasing. https://nodejs.org/docs/latest/api/crypto.html#cryptorandomuuidv7options
- `crypto.getRandomValues()` from `node:crypto` is not compliant with the Web Crypto spec. For web-compatible code use `crypto.webcrypto.getRandomValues()`, which throws above 65,536 bytes and rejects float arrays. https://nodejs.org/docs/latest/api/webcrypto.html#cryptogetrandomvaluestypedarray

## Hashing and MACs

- Use the one-shot `crypto.hash(alg, data)` for small in-memory data (about 5 MB or less), `createHash` for large or streamed data. Output defaults to `'hex'`. Stable since v24.13.1. https://nodejs.org/docs/latest/api/crypto.html#cryptohashalgorithm-data-options

```js
const digest = crypto.hash('sha256', payload); // hex string, UTF-8 input
```

- A `Hash` cannot be reused after `digest()`. `hash.copy()` gives a rolling hash and throws after `digest()`. https://nodejs.org/docs/latest/api/crypto.html#hashcopyoptions
- Call `hmac.digest()` once: a second call returns an empty buffer today and will throw. https://nodejs.org/docs/latest/api/deprecations.html#DEP0206
- SHAKE-128/256 digests need an explicit `outputLength` (runtime deprecation v25.0.0). https://nodejs.org/docs/latest/api/deprecations.html#DEP0198
- An HMAC key should not exceed the hash block size. https://nodejs.org/docs/latest/api/crypto.html#cryptogeneratekeytype-options-callback
- `crypto.createMac()` and `Mac` (added in v26.9.0, release candidate) expose provider MACs (HMAC, CMAC, GMAC, KMAC, BLAKE2). Use `mac.end()` or `mac.final()`, never both. A finalized `Mac`, even a failed one, throws `ERR_CRYPTO_MAC_FINALIZED`. https://nodejs.org/docs/latest/api/crypto.html#class-mac

## Password KDFs

- Prefer the async KDF variants so the threadpool does the work. https://nodejs.org/docs/latest/api/crypto.html#cryptoscryptpassword-salt-keylen-options-callback
- `pbkdf2`: `iterations` as high as possible, random salt of at least 16 bytes, explicit `digest` (required). https://nodejs.org/docs/latest/api/crypto.html#cryptopbkdf2password-salt-iterations-keylen-digest-callback
- `scrypt` defaults: `N` 16384, `r` 8, `p` 1, `maxmem` 32 MiB. It errors when about `128 * N * r > maxmem`. https://nodejs.org/docs/latest/api/crypto.html#cryptoscryptpassword-salt-keylen-options-callback
- `crypto.argon2()` (added v24.7.0) requires `message`, `nonce` (at least 8 bytes, 16 random recommended), `parallelism`, `tagLength`, `memory` and `passes`. Never store the optional `secret` (pepper) next to the derived key. https://nodejs.org/docs/latest/api/crypto.html#cryptoargon2algorithm-parameters-callback
- Call `normalize()` on user passphrases before deriving: Node does not normalize, and composed and decomposed forms give different keys. https://nodejs.org/docs/latest/api/crypto.html#using-strings-as-inputs-to-cryptographic-apis

## Strings and algorithm strength

- Use random bytes for secret keys, not strings: arbitrary bytes are not valid UTF-8, so string keys lose entropy and round-tripping replaces sequences with U+FFFD. Never treat cipher, hash, signature or KDF output as text. https://nodejs.org/docs/latest/api/crypto.html#using-strings-as-inputs-to-cryptographic-apis
- MD5 and SHA-1 are unacceptable where collision resistance matters. RSA, DSA and DH keys need at least 2048 bits, ECDSA/ECDH curves at least 224 bits. The module still allows weak choices. https://nodejs.org/docs/latest/api/crypto.html#support-for-weak-or-compromised-algorithms
- DH groups `modp1`, `modp2`, `modp5` have keys under 2048 bits and are not recommended. https://nodejs.org/docs/latest/api/crypto.html#support-for-weak-or-compromised-algorithms
- Legacy-provider algorithms are off by default. https://nodejs.org/docs/latest/api/crypto.html#support-for-weak-or-compromised-algorithms

## Availability

- Node can be built without crypto. `require('node:crypto')` then throws (catch it). A static `import` can only be caught by a preloaded `uncaughtException` handler, so use `await import()` in `try`. https://nodejs.org/docs/latest/api/crypto.html#determining-if-crypto-support-is-unavailable

## Version-flagged APIs

| API | Status | Source |
| :-- | :-- | :-- |
| `crypto.encapsulate()` / `decapsulate()` (KEM, incl. ML-KEM) | added v24.7.0 | https://nodejs.org/docs/latest/api/crypto.html#cryptodecapsulatekey-ciphertext-callback |
| `crypto.diffieHellman()` accepts key data, not only `KeyObject` | changed in v26.1.0 | https://nodejs.org/docs/latest/api/crypto.html#cryptodiffiehellmanoptions-callback |
| `crypto.parsePKCS12(bundle)` | added in v26.10.0 | https://nodejs.org/docs/latest/api/crypto.html#cryptoparsepkcs12bundle-options |
| `crypto.getMacs()` | added in v26.9.0, release candidate | https://nodejs.org/docs/latest/api/crypto.html#cryptogetmacs |
| `crypto.hash()` with OpenSSL-provider algorithms, cSHAKE options | added in v26.9.0, cSHAKE needs OpenSSL 4.0+ | https://nodejs.org/docs/latest/api/crypto.html#cryptohashalgorithm-data-options |

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `crypto.createCipher()` (removed) | `scrypt`/`pbkdf2` with random salt, then `createCipheriv()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0106 |
| `pbkdf2` without digest | pass the digest (throws) | https://nodejs.org/docs/latest/api/deprecations.html#DEP0009 |
| `crypto.fips` | `crypto.getFips()` / `crypto.setFips()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0093 |
| `crypto.setEngine()`, `clientCertEngine`, `privateKeyEngine` | OpenSSL 3 providers | https://nodejs.org/docs/latest/api/deprecations.html#DEP0183 |
| `new crypto.Hash()`, `new crypto.Hmac()` | `crypto.createHash()`, `crypto.createHmac()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0179, https://nodejs.org/docs/latest/api/deprecations.html#DEP0181 |
| `ecdh.setPublicKey()` | nothing, it is not useful | https://nodejs.org/docs/latest/api/deprecations.html#DEP0031 |
| `new crypto.Certificate()` (SPKAC) | static `Certificate` methods | https://nodejs.org/docs/latest/api/crypto.html#legacy-api |
