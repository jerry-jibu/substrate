# ateapi/internal/ateletauth

## Overview

Authenticates RPCs that arrive from an atelet, for the ateapi services served
only to atelet (`ateletauth.go` package doc).

## Interface

- `Authenticate(ctx, ateletSPIFFEID)` (`ateletauth.go`) - verifies the RPC arrived over mTLS from an atelet presenting the expected SPIFFE ID and returns the identity its certificate asserts.
- `Caller` (`ateletauth.go`) - `{PodName, NodeName}`, read from the certificate's PodIdentity extension.

## Invariants

- The TLS layer has already verified the chain against the pod-identity CA, so the extensions read here are trustworthy: only the pod-identity signer can mint a certificate carrying a given pod's node name (`ateletauth.go` `Authenticate`).
- Only atelet may call these RPCs; every other holder of a valid pod-identity certificate, including actor workloads, is rejected (`ateletauth.go` `Authenticate`).
- Rejection is uniform: the caller learns it is not atelet and nothing about why (`ateletauth.go` `denied`).

## Side Effects

- ["None"]

## Verification

- Run: `go test ./cmd/ateapi/internal/ateletauth/...`
