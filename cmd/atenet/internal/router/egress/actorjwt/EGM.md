# actorjwt

## Overview

Mints the actor JWTs that egress policies inject, through ate-api-server's
`ControlClient.MintActorJwt`, with caching and singleflight.

## Interface

- `actorjwt.go:New` - returns a `Minter` that mints through `client`.
- `actorjwt.go:(*Minter).Token` - returns a JWT for the actor ref, bound to the source's audiences and lifetime.

## Invariants

- Each token is reused until a third of its lifetime remains (`reuseFor` computation in `actorjwt.go:mint`).
- Tokens are cached by actor name, so an actor recreated under the same name can get its predecessor's token until the refresh point (`actorjwt.go:Minter` doc).
- The mint outlives its caller: the flight runs on `context.WithoutCancel`, so a leader going away does not fail joined callers (`actorjwt.go:(*Minter).Token`).
- Cache keys are the deterministic proto encoding of the mint request, stable within one binary (`actorjwt.go:tokenKey`).

## Side Effects

- Calls `ateapi.ControlClient.MintActorJwt` over the network; holds an in-process expiring token cache and a singleflight group.

## Verification

- Run: `go test ./cmd/atenet/internal/router/egress/actorjwt/...`
