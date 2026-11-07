# gcsissuer

## Overview
Mints GCS tokens downscoped to an actor's grants ("Package gcsissuer", `issuer.go`). The GCS half of the control plugin's credential exchange: `actorscope` decides what an actor may touch, this package turns that into a credential GCS enforces. The source credential is a token for the snapshot transfer service account.

## Interface
- `issuer.go:Issuer` implements `exchange.Issuer`; built with `issuer.go:New(ctx context.Context, account string, opts ...Option)`.
- Options: `issuer.go:WithIAMCredentialsOptions`, `WithTokenSource`, `WithHTTPClient`, `WithClock`, `WithCacheSize(n int)`.

## Invariants
- Issued tokens carry Credential Access Boundary rules derived from the actor's grants (`gcsboundary.FromGrants`), so they are enforceably scoped to the actor's snapshot prefixes.
- Issued tokens are cached per actor (`issuer.go:WithCacheSize`), and `exchange.Issuer.Issue`'s `force` flag bypasses that cache.

## Side Effects
- Calls Google IAM Credentials (or the configured `oauth2.TokenSource` / HTTP client) to exchange the service account token for a downscoped one (`issuer.go:New` options).

## Verification
- Run: `go test ./internal/objectstoreplugin/gcsissuer/...`
