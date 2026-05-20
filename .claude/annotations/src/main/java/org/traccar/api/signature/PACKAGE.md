# `api/signature/` — Cryptographic signing and token management

3 classes implementing ECDSA-based token signing. The EC key pair is persisted in DB; tokens are custom-signed JSON blobs (not standard JWT).

## File index

| File | One-liner | Annotation |
|---|---|---|
| `CryptoManager.java` | EC key pair manager (secp256r1); sign/verify ECDSA; persists in tc_keystore | [CryptoManager.java.md](CryptoManager.java.md) |
| `TokenManager.java` | Generates/verifies Bearer tokens; revocation via tc_revoked_tokens | [TokenManager.java.md](TokenManager.java.md) |
| `KeystoreModel.java` | `@StorageName("tc_keystore")` model for public+private key DER bytes | [KeystoreModel.java.md](KeystoreModel.java.md) |

## Token format (not JWT)

```
generate:  JSON{i,u,e}  →  CryptoManager.sign  →  [sigLen][sig][data]  →  base64url
verify:    base64url  →  CryptoManager.verify  →  JSON{i,u,e}  →  check expiry + revocation
```

## Key rotation

1. Delete the row in `tc_keystore`.
2. Restart the server — new key pair generated on first token operation.
3. All existing tokens are immediately invalid (new public key cannot verify old signatures).

## Dependency graph

```
TokenManager → CryptoManager → tc_keystore (KeystoreModel)
TokenManager → tc_revoked_tokens (RevokedToken)
OidcSessionManager → CryptoManager (for OIDC signing key)
