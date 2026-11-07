# principal

## Overview

Carries the authenticated principal's identity through a request context:
the small shared type the mTLS and JWT authenticators both produce.

## Interface

- `principal.go:PrincipalInfo` - `ID`, `Kind`, `Issuer` of an authenticated principal.
- `principal.go:KindMTLS` (`mtls`) and `principal.go:KindJWT` (`jwt`) - kind values named after the authentication method.
- `principal.go:FromContext`/`InjectContext` - read/write the principal in a context under a private key.
- Subdirectories: none.

## Invariants

- ["None"]

## Side Effects

- ["None"]

## Verification

- Run: `go test ./internal/principal/...`
