# exchange

## Overview
Serves the objectstore credential exchange ("Package exchange", `exchange.go`). The node plugin presents the worker-for-actor JWT atelet gave it and gets back a storage access token scoped to that actor's snapshot prefixes. The provider-neutral parts (actorauthz, actorscope) stay out of this package; this package and its proto are provider-shaped, and `Issuer` is the seam between backends.

## Interface
- `exchange.go:Server` implements `objectstoreexchange.CredentialExchange` (`internal/proto/objectstoreexchangepb`); RPC `exchange.go:Server.ExchangeActorJwt`.
- `exchange.go:New(authorizer Authorizer, issuer Issuer) *Server`.
- Seams: `exchange.go:Issuer` (mints a credential for an actor's grants; errors with a non-Unavailable gRPC code pass through unchanged), `exchange.go:Authorizer` (implemented by `actorauthz`; its status errors return as-is).
- `exchange.go:DefaultPort` (8445) is the port the control sidecar serves and the node plugin dials by default.

## Invariants
- A credential is issued only for a JWT the Authorizer accepts (`exchange.go:ExchangeActorJwt` doc).
- Issuer errors map deterministically: a non-Unavailable gRPC status passes through, a wrapped context error keeps its code, everything else becomes Unavailable (`exchange.go:Issuer` doc).

## Side Effects
- Serves the `objectstoreexchange.CredentialExchange` gRPC service over the ate-api-server Service.
- Delegates token minting to the configured `Issuer` (GCS: `gcsissuer`).

## Verification
- Run: `go test ./internal/objectstoreplugin/exchange/...`
