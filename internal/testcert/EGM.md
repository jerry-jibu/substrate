# testcert

## Overview

Test helper: issues origin TLS certificates for e2e tests of the gateway
data path. Only the public CA is handed to the caller; the signing key
stays in the test process.

## Interface

- `servertls.go:ServerTLS` - a key triple: `RootCA`, `Certificate`, `PrivateKey`, all PEM.
- `servertls.go:NewServerTLS` - generates a throwaway CA (`localca.GenerateCA`, ECDSA P-256, 24h) and issues a certificate for the given Service IP.
- `servertls.go:ServerTLSWithCA` - issues an IP (required) plus optional DNS-name certificate from a CA already trusted by the client or the MITM gateway's upstream connection; leaf valid 1 hour, serverAuth.
- Subdirectories: none.

## Invariants

- Testing-only package; requires `*testing.T` and `t.Helper()` discipline.
- The origin certificate requires a Service IP (`servertls.go:ServerTLSWithCA` fatals on nil).

## Side Effects

- Generates ECDSA keys and certificates in-process; no filesystem or network writes.

## Verification

- Run: `go test ./internal/testcert/...`
