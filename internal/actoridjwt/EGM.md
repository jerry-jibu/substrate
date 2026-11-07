# actoridjwt

## Overview
Claim types for actor identity JWTs. Defines the two token kinds (an actor's own token and a Substrate component acting on the actor's behalf), their subject prefixes and claim names, and conversion between internal `Claims` and the JSON wire form.

## Interface
- `actoridjwt.go:Kind` - `KindActor`, `KindWorkerForActor`; `SubjectPrefix()` and `ClaimName()` return the `sub` prefix and the JWT claim name for the kind.
- `actoridjwt.go:Claims`, `actoridjwt.go:SubstrateClaims` - internal claim model; `SubstrateClaims.Node` names the node holding the actor.
- `actoridjwt.go:WireClaims`, `actoridjwt.go:WireSubstrateClaims` - JSON wire form with claims under `actor.ate.dev` or `worker-for-actor.ate.dev`; `WireClaims.Substrate()` returns the kind and claims, `ok` false when the token carries neither claim or both.
- `actoridjwt.go:ClaimsToWire` - converts `Claims` to `WireClaims`.
- `issuer.go:ValidateIssuer` - validates that an issuer is `https://` plus host and optional port, nothing after.

## Invariants
- An actor's own token carries no node: `ClaimsToWire` errors when `KindActor` has a non-empty `Node` (`actoridjwt.go:ClaimsToWire`).
- A kind other than `KindActor` or `KindWorkerForActor` is rejected (`actoridjwt.go:ClaimsToWire`).
- A token carrying both substrate claims or neither reports `ok == false` from `WireClaims.Substrate()` (`actoridjwt.go:WireClaims.Substrate`).

## Side Effects
- None. Pure data conversion and validation.

## Verification
- Run: `go test ./internal/actoridjwt/...`
