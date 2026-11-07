# ate-idp-server

## Overview
Binary that serves the OpenID Connect discovery document and JWK set used to verify actor JWTs. It publishes the public keys of the authority pool in `--actor-id-jwt-pool`, rereading it at most once per minute, over HTTPS using the credential bundle in `--server-cred-bundle`.

## Interface
- `main.go:main` / `main.go:run` - CLI entry point; wires a `server.Server` into an `http.Server` with TLS.
- Flags (`main.go` var block): `--issuer` (required, must match ate-api-server's `--actor-jwt-issuer`), `--actor-id-jwt-pool` (required), `--server-cred-bundle` (required), `--listen-address` (default `:8443`), `--log-level`, `--drain-grace`, `--preview`.
- [internal/server](./internal/EGM.md) - HTTP handlers for discovery, JWKS, and health.

## Invariants
- `main.go:run` enforces that `--issuer`, `--actor-id-jwt-pool`, and `--server-cred-bundle` are set (`main.go:missingFlags`).
- TLS config requires `MinVersion` TLS 1.2 and loads the serving certificate through `credbundle.Loader` (`main.go:run`).
- On SIGINT/SIGTERM the server drains in-flight requests for at most `--drain-grace` before exiting (`main.go:run`).

## Side Effects
- Listens for HTTPS on `--listen-address` (`main.go:run`).
- Periodically rereads the pool file and the credential bundle from disk via `localjwtauthority.NewRefreshingPool` and `credbundle.Loader` (`main.go:run`).

## Verification
- Run: `go test ./cmd/ate-idp-server/...`
