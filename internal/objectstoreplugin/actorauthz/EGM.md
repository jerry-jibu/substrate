# actorauthz

## Overview

The provider-neutral checks every provider's credential exchange must apply to a worker-for-actor JWT before it issues any credential: verify the JWT, look the actor up, check the actor is still on the token's node, and derive the actor's storage grants. Imports no cloud SDK.

## Interface

- `actorauthz.go:Audience` - `"objectstore-plugin.ate.dev"`, the audience every exchange requires on the worker-for-actor JWT.
- `actorauthz.go:NewVerifier(issuer, client)` and `actorauthz.go:StartVerifier(ctx, issuer, caFile) (*oidcjwt.Verifier, *http.Client, error)` - the verifier, accepting only this audience, with keys fetched via OIDC discovery.
- `actorauthz.go:Verifier` / `actorauthz.go:Lookup` interfaces (satisfied by `oidcjwt.Verifier` and `actorlookup.Lookup`).
- `actorauthz.go:New(v Verifier, l Lookup) *Authorizer` and `actorauthz.go:Authorizer.Authorize(ctx, jwt) (*Decision, error)`.
- `actorauthz.go:Decision` - `ActorUID`, `Grants []actorscope.Grant`, `Expiry` (the token's exp; a credential issued from it must not outlive it).

## Invariants

- Only a worker-for-actor token (`actoridjwt.KindWorkerForActor`) is accepted; an actor's own token has the same shape and must not yield credentials for its own snapshots (`actorauthz.go:Authorizer.Authorize`).
- The claims must carry atespace, actor name, UID, and minting node (`actorauthz.go:Authorizer.Authorize`).
- An actor in a state outside `resources.TransferStates`, or whose `status.assigned_node` differs from the token's node, is refused: the token can outlive the mint-time node check by up to an hour (`actorauthz.go:actorStillOnNode`).
- A recreated actor is caught by the UID check in `actorlookup` (`actorlookup.ErrStale` mapped to NotFound) (`actorauthz.go:Authorizer.Authorize`).
- An actor with no snapshot location to grant is refused (`actorauthz.go:Authorizer.Authorize`).
- A `.svc` issuer host is trusted only through the CA file; any other host through system roots or the CA file (`actorauthz.go:issuerRoots`).
- At startup, a certificate-verification failure from the issuer fails the start; any other fetch error is logged and the exchange serves, retrying later (`actorauthz.go:checkIssuerCertificate`).
- Every error returned is a gRPC status error the exchange can return as is (`actorauthz.go:Authorizer.Authorize`).

## Side Effects

- Fetches the issuer's OIDC signing keys over HTTPS, caching and refreshing them (`actorauthz.go:NewVerifier`, `actorauthz.go:StartVerifier`); the caller closes the client's idle connections when done.

## Verification

- Run: `go test ./internal/objectstoreplugin/actorauthz/...`
