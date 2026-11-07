# atenet

## Overview
Shared atenet data-plane constants and helpers. Holds the HTTP/2 header that carries the target actor so dataplane configuration and metadata stay native to the wire form.

## Interface
- `headers.go:TargetActorHeader` - the `ate-target-actor` HTTP/2 header constant.
- `headers.go:ParseTargetActor` - parses and validates a `TargetActorHeader` value into a `resources.ActorRef`.

## Invariants
- The header value must parse into a valid `resources.ActorRef`; anything else is an error (`headers.go:ParseTargetActor`).

## Side Effects
- None. Pure header parsing.

## Verification
- Run: `go test ./internal/atenet/...`
