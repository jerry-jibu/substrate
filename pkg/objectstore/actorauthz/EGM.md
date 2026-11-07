# objectstore/actorauthz

## Overview
Public wrapper around Substrate's GCS credential-exchange checks, for out-of-tree object-store providers that serve an exchange of their own. `Authorize` verifies the worker-for-actor JWT atelet sends with a transfer, looks the actor up in ate-api-server, and returns the storage prefixes a credential must be limited to. The heavy lifting lives in `internal/objectstoreplugin`; this package re-exports it so external providers run the same code while only this API is public.

## Interface
- `New(ctx, cfg) (*Authorizer, error)` and `Authorizer.Close()` (`actorauthz.go:New`, `actorauthz.go:Authorizer`)
- `Authorizer.Authorize(ctx, jwt) (*Decision, error)`; safe for concurrent use (`actorauthz.go:Authorize`, `actorauthz.go:Authorizer` doc comment)
- `Config` (`Issuer`, `IssuerCAFile`, `APIServer`) and `APIServerConfig` (`Address`, `CAFile`, `ServerName`, `ClientCredBundle`) (`actorauthz.go:Config`, `actorauthz.go:APIServerConfig`)
- `Decision` with `ActorUID`, `Grants`, `Expiry`; `Grant` with `Prefix` and `Access`; `Access` values `ReadOnly`/`ReadWrite` (`actorauthz.go:Decision`, `actorauthz.go:Grant`, `actorauthz.go:Access`)
- Constants `Audience` (the objectstore v1 actor_jwt audience) and `MaxJWTSize` (8 KiB) (`actorauthz.go:Audience`, `actorauthz.go:MaxJWTSize`)

## Invariants
- Only a worker-for-actor token is accepted, never an actor's own; every error is a gRPC status with documented codes (InvalidArgument, Unauthenticated, PermissionDenied, NotFound, FailedPrecondition, Unavailable, Internal, Canceled/DeadlineExceeded) (`actorauthz.go:Authorize` doc comment).
- A credential issued from a `Decision` must not outlive `Decision.Expiry` (`actorauthz.go:Decision` field doc).
- `Grant.Prefix` ends in `/` so string prefix matches cannot reach a sibling whose name starts the same way; parse it with `url.Parse` before comparing to object names (`actorauthz.go:Grant` field doc).
- The issuer's keys come from OpenID Connect discovery at `<Issuer>/.well-known/openid-configuration`; `IssuerCAFile` is required and is the only trusted bundle for `.svc` hosts (`actorauthz.go:Config` field docs).
- The exchange must run in the ate-api-server pod, present that pod's identity certificate, serve over TLS, admit only atelet, and call `Authorize` for every request; the JWT is a bearer token (`actorauthz.go` package doc).
- All `Config` fields the package checks are required: empty `Issuer`, `IssuerCAFile`, `APIServer.Address`, `CAFile`, `ServerName`, or `ClientCredBundle` fail `New` (`actorauthz.go:New`).

## Side Effects
- HTTPS fetch of the issuer's JWKS during `New` and again from `Authorize` when keys are unavailable (`actorauthz.go:New`, `actorauthz.go:Authorize` doc).
- mTLS gRPC connection to ate-api-server over loopback for actor lookups, created lazily and closed by `Close` (`actorauthz.go:APIServerConfig` field doc, `actorauthz.go:Close`).
- Re-reads `IssuerCAFile` on each handshake when it changes, so certificate rotation needs no restart; a failed re-read keeps the last bundle (`actorauthz.go:IssuerCAFile` field doc).

## Verification
- Run: `go test ./pkg/objectstore/actorauthz/...` from the repo root.
