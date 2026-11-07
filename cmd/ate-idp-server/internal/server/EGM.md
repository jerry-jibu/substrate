# ate-idp-server/internal/server

## Overview
Serves the actor JWT issuer's OpenID Connect discovery document and JWK set so relying parties can verify actor JWTs. Builds both documents from a `KeySource` on every request.

## Interface
- `server.go:KeySource` - interface supplying verification keys; `*localjwtauthority.RefreshingPool` is one implementation.
- `server.go:Server` - serves discovery at `GET /.well-known/openid-configuration`, the JWK set at `GET /openid/v1/jwks`, and `GET /healthz`.
- `server.go:New` - constructs a `Server` for an issuer URL that must be `https://<host>` (validated via `actoridjwt.ValidateIssuer`).
- `server.go:Register` - adds the three routes to an `http.ServeMux`.

## Invariants
- `server.go:New` fails if the issuer URL is invalid or the keys cannot be published once at construction time.
- `server.go:build` regenerates both documents from the `KeySource` on every request; if key reading fails, `server.go:serveDocument` answers HTTP 500.

## Side Effects
- Reads verification keys from the `KeySource` on every request (`server.go:build`).
- Writes HTTP responses and logs key-publish failures via `slog` (`server.go:serveDocument`).

## Verification
- Run: `go test ./cmd/ate-idp-server/internal/server/...`
