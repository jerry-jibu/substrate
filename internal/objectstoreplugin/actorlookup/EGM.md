# actorlookup

## Overview

Fetches the actor an exchange request names from ate-api-server. The control plugin runs as a sidecar in the ate-api-server pod, so it dials the loopback address and presents the pod identity certificate, which ate-api-server accepts as an mTLS client.

## Interface

- `lookup.go:Config` - `Address` (127.0.0.1:443 in the pod), `CAFile`, `ServerName`, `ClientCredBundle`; every file is re-read on each handshake.
- `lookup.go:New(cfg Config) (*Lookup, error)` - dials lazily, so a server that is not up fails the first lookup, not New.
- `lookup.go:Lookup.Actor(ctx, claims *actoridjwt.WireSubstrateClaims) (*ateapi.Actor, error)` - calls `GetActor` with the claims' atespace and name.
- `lookup.go:Lookup.Close()`; `lookup.go:ErrStale`.

## Invariants

- The claims must carry atespace, actor name, and UID (`lookup.go:Lookup.Actor`).
- A returned actor whose UID differs from the claims' UID yields `ErrStale`: the actor was deleted and recreated since the JWT was minted (`lookup.go:ErrStale`, `lookup.go:Lookup.Actor`).
- gRPC status errors from ate-api-server (e.g. NotFound) are returned unchanged (`lookup.go:Lookup.Actor`).
- The connection carries an otelgrpc stats handler so the exchange's trace continues into GetActor (`lookup.go:New`).

## Side Effects

- Holds one mTLS gRPC client connection to ate-api-server (`lookup.go:New`).

## Verification

- Run: `go test ./internal/objectstoreplugin/actorlookup/...`
