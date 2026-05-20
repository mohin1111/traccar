# KeystoreModel.java

**Role:** JPA-less model for the `tc_keystore` table. Holds the raw DER-encoded public and private key bytes for the server's EC key pair. Single row expected in the table.
**Fits in:** Read and written by [[CryptoManager.java]] `loadOrGenerate()`. No resource exposes this directly.
**Read next:** [[CryptoManager.java]] (the only consumer)

## Public API

- `getPublicKey()` / `setPublicKey(byte[])` — X.509 DER-encoded EC public key.
- `getPrivateKey()` / `setPrivateKey(byte[])` — PKCS8 DER-encoded EC private key.

## Gotchas / non-obvious

- `@StorageName("tc_keystore")` — maps to the keystore table. `BaseModel` provides the `id` field, but the table is intended to have only one row (id=1).
- Byte arrays stored as BLOBs/BYTEA depending on DB type. No Base64 encoding at the model level — done by JDBC layer.
