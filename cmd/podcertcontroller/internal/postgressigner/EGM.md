# podcertcontroller/internal/postgressigner

## Overview
Signer `postgres.podcert.ate.dev/identity`. Issues PostgreSQL login certificates to the ate-api-server service account, restricted to the two bundled application database logins.

## Interface
- `postgressigner.go:Name` - const `"postgres.podcert.ate.dev/identity"`.
- `postgressigner.go:UsernameAnnotation` - const `postgres.podcert.ate.dev/username`; selects the bundled database login.
- `postgressigner.go:Impl` / `postgressigner.go:NewImpl` - signer implementation of `signercontroller.SignerImpl`, scoped to one namespace.
- `postgressigner.go:DesiredClusterTrustBundles` - PEM bundle of CA trust anchors named `CTBPrefix + "primary-bundle"`.
- `postgressigner.go:MakeCert` - issues a login certificate for the requested username.

## Invariants
- Only `<postgres-client-namespace>/ate-api-server` may request PostgreSQL login certificates; anything else is denied (`postgressigner.go:MakeCert`).
- Annotations are untrusted input: the request must contain exactly one annotation naming `postgressetup.OwnerUser` or `postgressetup.ReadWriteUser`; the administrator login is never issued (`postgressigner.go:MakeCert`).
- Requests for a different signer name are rejected (`postgressigner.go:MakeCert`).

## Side Effects
- Denies unauthorized requests by writing a denied condition through PCR status (`postgressigner.go:MakeCert` -> `deny`).
- Reads CA pool state via `localca.Pool`; certificate lifetime defaults to 24h, shortened by `MaxExpirationSeconds`.

## Verification
- Run: `go test ./cmd/podcertcontroller/internal/postgressigner/...`
