# credential

## Overview

Holds the credential type the exchange hands out and its issuers produce. A leaf package so an issuer does not depend on the handler that serves it.

## Interface

- `credential.go:Token` - `AccessToken string` and `Expiry time.Time`.
- `credential.go:MinServeLifetime` - 20 minutes.

## Invariants

- `MinServeLifetime` is the least remaining lifetime a cached Token is served with, on both sides of the exchange, so a transfer never starts on a token about to expire; there is no mid-transfer refresh. A token with less left is still returned to the caller that asked for it, but not cached (`credential.go:MinServeLifetime` doc).

## Side Effects

- ["None"]

## Verification

- Run: `go test ./internal/objectstoreplugin/credential/...`
