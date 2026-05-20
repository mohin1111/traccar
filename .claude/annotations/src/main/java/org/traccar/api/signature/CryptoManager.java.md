# CryptoManager.java

**Role:** Manages a persistent EC key pair (secp256r1 / P-256) stored in `tc_keystore`. Used to sign and verify Bearer tokens and to sign OIDC ID tokens. Singleton with volatile double-checked locking for lazy key initialization.
**Fits in:** Used by [[TokenManager.java]] (token signing/verification) and [[security/OidcSessionManager.java]] (OIDC ID token signing key).
**Read next:** [[TokenManager.java]] (calls `sign`/`verify`), [[KeystoreModel.java]] (the DB storage entity)

## Public API

- `sign(byte[] data)` (line 49) — signs data with SHA256withECDSA; prepends a 1-byte length prefix + signature block before the data. Returns combined byte array.
- `verify(byte[] data)` (line 61) — verifies the combined format; returns the original data on success; throws `SecurityException("Invalid signature")` on failure.
- `getKeyPair()` (line 74) — lazy-load from `tc_keystore`; generates and persists new P-256 key pair if none found.

## Key flows

### Token format (sign output)
`[1 byte: signature length][N bytes: ECDSA signature][M bytes: original data]`. This self-describing format makes `verify` stateless — no separate header needed.

### Key persistence (`loadOrGenerate`, lines 88-106)
1. Query `tc_keystore` (single row).
2. If found: reconstruct `PublicKey` (X.509) + `PrivateKey` (PKCS8).
3. If not found: generate secp256r1 `KeyPair`, persist to `tc_keystore`.

## Gotchas / non-obvious

- **Single row in `tc_keystore`** — only one key pair supported. Key rotation requires deleting the row; all existing tokens become invalid immediately.
- **Volatile + synchronized double-checked locking** (lines 75-85) — correct pattern for lazy singleton in Java; `volatile keyPair` prevents instruction reordering.
- `OidcSessionManager` also caches the key pair as `ECKey` — both caches must be cleared (server restart) on key rotation.

## Line index

- 49-58 — `sign` (prepend signature + data)
- 61-71 — `verify` (parse format + verify signature)
- 74-86 — `getKeyPair` (lazy + thread-safe)
- 88-106 — `loadOrGenerate` (DB load or P-256 generate + persist)
