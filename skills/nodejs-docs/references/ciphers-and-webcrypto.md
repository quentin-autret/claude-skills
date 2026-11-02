# Ciphers, AEAD and Web Crypto

## Contents

- AEAD tag flow
- CCM
- Key wrap, SIV and GCM-SIV
- Padding and IVs
- Web Crypto
- Old patterns

## AEAD tag flow

- For GCM, CCM, OCB, SIV, GCM-SIV and chacha20-poly1305: call `setAAD()` before `update()`, and `getAuthTag()` only after `final()`. On decryption call `setAuthTag()` once, before `final()` for GCM, OCB and chacha20-poly1305, before `update()` for CCM, SIV and GCM-SIV. https://nodejs.org/docs/latest/api/crypto.html#deciphersetauthtagbuffer-encoding
- A missing tag or tampered ciphertext makes `final()` throw: discard any plaintext already produced. https://nodejs.org/docs/latest/api/crypto.html#deciphersetauthtagbuffer-encoding
- A GCM tag shorter than 16 bytes requires `authTagLength` in `createDecipheriv()`. Without it this is an error since v26.0.0 (end-of-life of DEP0182, a runtime deprecation since v23.0.0). Only 128, 120, 112, 104, 96, 64 and 32-bit tags are valid. https://nodejs.org/docs/latest/api/deprecations.html#DEP0182 , https://nodejs.org/docs/latest/api/deprecations.html#DEP0090

## CCM

- `authTagLength` is required at creation (4 to 16, even). Nonce is 7 to 13 bytes. `setAAD()` needs `plaintextLength`. Call `update()` exactly once, never `write()`/`end()`/`pipe()`, and always call `final()`. The ciphertext does not include the tag. https://nodejs.org/docs/latest/api/crypto.html#ccm-mode

```js
const cipher = createCipheriv('aes-192-ccm', key, nonce, { authTagLength: 16 });
cipher.setAAD(aad, { plaintextLength: Buffer.byteLength(plaintext) });
const ciphertext = cipher.update(plaintext, 'utf8');
cipher.final();
const tag = cipher.getAuthTag();

const decipher = createDecipheriv('aes-192-ccm', key, nonce, { authTagLength: 16 });
decipher.setAuthTag(tag);
decipher.setAAD(aad, { plaintextLength: ciphertext.length });
const received = decipher.update(ciphertext, null, 'utf8');
try { decipher.final(); } catch (err) { throw new Error('Authentication failed!', { cause: err }); }
```
https://nodejs.org/docs/latest/api/crypto.html#ccm-mode

## Key wrap, SIV and GCM-SIV

- `AES-WRAP` / `AES-WRAP-PAD`: one `update()` with the complete non-empty value, then `final()`. Never stream it: splitting produces several wraps. https://nodejs.org/docs/latest/api/crypto.html#aes-key-wrap-modes
- SIV and GCM-SIV tags are 16 bytes. An AES-SIV key is double size (`aes-256-siv` needs 64 bytes) and takes no IV (`null`). Each `setAAD()` is one ordered input (max 126), and a nonce goes last. AES-GCM-SIV needs a 12-byte IV. https://nodejs.org/docs/latest/api/crypto.html#siv-and-gcm-siv-modes

## Padding and IVs

- PKCS padding is automatic. With `setAutoPadding(false)` the input must be a multiple of the block size or `final()` throws. https://nodejs.org/docs/latest/api/crypto.html#ciphersetautopaddingautopadding
- IV requirements depend on the algorithm (unique and unpredictable, unique only, fixed or none). IVs are not secret and travel with the ciphertext. https://nodejs.org/docs/latest/api/crypto.html#cryptocreatecipherivalgorithm-key-iv-options

## Web Crypto

- Reach it through `globalThis.crypto` (or `require('node:crypto').webcrypto`). Default to it for code shared with browsers. https://nodejs.org/docs/latest/api/webcrypto.html

```js
const { subtle } = globalThis.crypto;
const key = await subtle.generateKey({ name: 'HMAC', hash: 'SHA-256', length: 256 }, true, ['sign', 'verify']);
const mac = await subtle.sign({ name: 'HMAC' }, key, new TextEncoder().encode('message'));
```
https://nodejs.org/docs/latest/api/webcrypto.html

- Format matrix: HKDF, PBKDF2 and Argon2 keys are import-only. AES and HMAC use `jwk`, `raw`, `raw-secret`. EC, Ed and X curves use `spki`, `pkcs8`, `jwk`, `raw`, `raw-public`. RSA supports only `spki`, `pkcs8`, `jwk`. ML-DSA and ML-KEM add `raw-seed`. https://nodejs.org/docs/latest/api/webcrypto.html#key-formats
- `subtle.digest()` core algorithms are SHA-1/256/384/512. https://nodejs.org/docs/latest/api/webcrypto.html#crypto-operation-apis
- "Modern Algorithms" are in active development (Stability 1.1, added across v24.7.0 to v26.10.0): AES-OCB, ChaCha20-Poly1305, Argon2, SHA-3, cSHAKE, KMAC, TurboSHAKE, ML-DSA, ML-KEM, hybrid ML-KEM, `subtle.encapsulate*`/`decapsulate*`, `getPublicKey`, `SubtleCrypto.supports()`. KangarooTwelve (KT128/256) is added in v25.9.0. Many need OpenSSL 3.0, 3.2 or 3.5. https://nodejs.org/docs/latest/api/webcrypto.html#modern-algorithms-in-the-web-cryptography-api
- Ed448 and X448 ("Secure Curves") are in active development. Ed25519 and X25519 are core. https://nodejs.org/docs/latest/api/webcrypto.html#secure-curves-in-the-web-cryptography-api

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `createCipher(alg, password)` | derive a key with `scrypt`, then `createCipheriv(alg, key, iv)` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0106 |
| short GCM tag without `authTagLength` | pass `authTagLength` (error since v26.0.0) | https://nodejs.org/docs/latest/api/deprecations.html#DEP0182 |
| `CryptoKey` passed to `createCipheriv`, `createHmac`, `sign`... | avoid it (runtime deprecation since v26.0.0, will throw in a future version) | https://nodejs.org/docs/latest/api/deprecations.html#DEP0203 |
| `require('node:crypto').getRandomValues` (not spec-compliant) | `globalThis.crypto.getRandomValues` for web-compatible code | https://nodejs.org/docs/latest/api/crypto.html#cryptogetrandomvaluestypedarray |
