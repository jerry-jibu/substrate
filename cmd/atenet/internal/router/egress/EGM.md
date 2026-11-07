# egress

## Overview
The egress direction of the atenet-router Envoy ext_proc server. `egress.go:Handler` enforces per-actor EgressPolicy on connections leaving a sandbox, injects credentials out of band, and selects the credential provider for the dialed host.

## Interface
- `egress.go:Handler`, built with `egress.go:New(apiClient ateapi.ControlClient, loadActorIdentityRoots func() (*x509.CertPool, error), policyCacheTTL time.Duration, providers map[string]credprovider.CredentialProviderClient)`; implements `Direction() extproc.Direction` (returns `extproc.DirectionEgress`) and `HandleRequestHeaders`.
- `policycache.go:DefaultPolicyCacheTTL` (10s) bounds how long a fetched policy stays usable.
- `provider.go:DialProvider(ctx, ProviderDialConfig)` dials a credential provider plugin; `provider.go:ProviderName` validates provider names.
- `actorjwt.Minter` (child `./actorjwt/EGM.md`): `actorjwt.go:Minter.Token` mints actor JWTs through the ateapi Control API for the egress sidecar.
- Sibling handlers: `./ingress/EGM.md` (actor ingress), `./extproc/EGM.md` (ext_proc mux).

## Invariants
- Egress policy enforcement is deny-by-default: traffic without an accepted policy is refused (`egress.go:Handler.HandleRequestHeaders`).
- Cached policies expire after `policycache.go:DefaultPolicyCacheTTL`; the TTL bounds how long a policy change lags.

## Side Effects
- Fetches actor EgressPolicies from ate-api-server (`ateapi.ControlClient`) and caches them in memory.
- Dials credential provider plugins over gRPC (`provider.go:DialProvider`) and hands provider-injected credentials to Envoy.
- Loads the actor identity CA roots for upstream TLS verification (`egress.go:New`).

## Verification
- Run: `go test ./cmd/atenet/internal/router/egress/...`
