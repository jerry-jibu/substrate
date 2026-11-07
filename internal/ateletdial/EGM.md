# ateletdial

## Overview
Dial helpers for ateworker processes connecting to the atelet socket on their own node, using the worker's Pod certificate for mutual TLS.

## Interface
- `ateletdial.go:TLSConfig` - builds a `*tls.Config` that authenticates this worker with its Pod certificate and accepts only the atelet on this worker's node; the caller passes atelet's SPIFFE ID in.
- `ateletdial.go:Dial` - opens a gRPC connection to the atelet socket; the caller closes it.

## Invariants
- `ateletSPIFFEID` names atelet's namespace, not the worker's, so it is a parameter rather than derived from the downward API (`ateletdial.go:TLSConfig`).
- A fresh connection picks up rotated worker credentials and re-verifies atelet (`ateletdial.go:Dial`).

## Side Effects
- Reads the credential bundle and trust bundle from disk via `internal/credbundle` loaders (`ateletdial.go:TLSConfig`).
- Connects to a Unix socket (`ateletdial.go:Dial`).

## Verification
- Run: `go test ./internal/ateletdial/...`
