# localjwtauthority

## Overview

A simple "CA" for JWTs: a Pool of signing authorities, one active for signing
and the rest trusted for verification, supporting seamless rotation. Built on
the standard library's crypto packages; used for actor JWT signing and key
discovery.

## Interface

- `Pool` (`localjwtauthority.go:Pool`): `SignJWT(*actoridjwt.Claims) (string, error)` and `VerificationKeys() ([]*VerificationKey, error)`.
- `VerificationKey` (`localjwtauthority.go:VerificationKey`): KeyID, Algorithm, PublicKey.
- `RefreshingPool` / `NewRefreshingPool(stateFile string)` (`localjwtauthority.go:NewRefreshingPool`): reloads pool state from disk so signing survives rotation without restarts.
- `ConcretePool` (`localjwtauthority.go:ConcretePool`) with rotation helpers `AddAuthority` (adds inactive), `Activate`, `RemoveAuthority`, `ActiveID` (`localjwtauthority.go:AddAuthority`).
- `Authority` (`localjwtauthority.go:Authority`): ID, Algorithm, SigningKey (`crypto.Signer`).
- `Marshal` / `Unmarshal` (`localjwtauthority.go:Marshal`): JSON serialization with PKCS#8 signing keys.
- `GenerateAuthority(algorithm, id string) (*Authority, error)` (`localjwtauthority.go:GenerateAuthority`): new RS256 (4096-bit) or ES256 key; empty id defaults to the key thumbprint.
- `GeneratePool(algorithm, id string) ([]byte, string, error)` (`localjwtauthority.go:GeneratePool`): serialized one-authority pool plus its ID.
- `JWKS(keys []*VerificationKey) ([]byte, error)` (`jwks.go:JWKS`): the RFC 7517 JWK set as JSON, sorted by key ID.
- `Thumbprint(pub crypto.PublicKey) (string, error)` (`thumbprint.go:Thumbprint`): base64url SHA-256 of the PKIX DER of an RSA or P-256 EC public key.

## Invariants

- `AddAuthority` requires a non-empty unique ID and adds it inactive, so relying parties learn the key before it signs (`localjwtauthority.go:AddAuthority`); `RemoveAuthority` refuses to remove the active authority (`localjwtauthority.go:RemoveAuthority`).
- When `ActiveForSigning` is empty the first authority signs, for backward compatibility (`localjwtauthority.go:ConcretePool.SignJWT`).
- Signing supports only RS256/RS384/RS512 (PKCS1v15) and ES256 with a P-256 key; other algorithms are rejected (`localjwtauthority.go:sign`).
- `RefreshingPool` reloads state at most once per minute, under its lock (`localjwtauthority.go:refreshIfNecessary`).
- `JWKS` rejects empty key sets, empty IDs, duplicate IDs, and algorithms that do not fit the key type (ES256/P-256, RS256/384/512 for RSA) (`jwks.go:JWKS`, `jwks.go:algorithmFits`).
- ES256 signatures use the fixed-width JOSE encoding (r and s padded to 32 bytes) (`localjwtauthority.go:sign`).
- `Thumbprint` supports only RSA and P-256 EC keys (`thumbprint.go:Thumbprint`).

## Side Effects

- `RefreshingPool` reads the pool state file from disk (`localjwtauthority.go:refreshIfNecessary`); key generation writes no external state.

## Verification

- Run: `go test ./internal/localjwtauthority/...`
