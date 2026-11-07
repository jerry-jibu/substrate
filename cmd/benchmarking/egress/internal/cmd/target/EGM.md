# target

## Overview

Endpoint of the egress scale test. The `target` cobra subcommand answers
every request with a small fixed body over plain HTTP and, when given a
certificate, HTTPS. One Deployment of it sits behind all of the test's
Services, which get distinct backend ports so connections on distinct
ClusterIPs cannot share a 4-tuple.

## Interface

- `target.go:Command` - cobra command `target`. Flags: `--listen` (default `:8080`), `--listen-ports` (range `A-B`), `--tls-listen`, `--tls-listen-ports`, `--tls-cert-file`, `--tls-key-file`, `--response-bytes` (default 16).
- `target.go:runTarget` - serves plain HTTP always, HTTPS only when a TLS address or port range is set; returns on context cancel or the first listener failure.
- `target.go:newHandler` - serves `GET /healthz` with `ok` and `GET /` with the fixed body.
- `target.go:portRange` - parses `A-B` or a single port; rejects ranges outside 1-65535.
- `target.go:openListeners`/`listenAll` - open all listeners or none, closing what was opened on error.
- Subdirectories: none.

## Invariants

- Every request gets HTTP 200 with exactly `--response-bytes` body bytes (`target.go:newHandler`).
- HTTPS requires both `--tls-cert-file` and `--tls-key-file`; TLS min version 1.2 (`target.go:newTLSConfig`).
- The first listener to fail stops all servers, like a signal (`target.go:runTarget` failed channel).

## Side Effects

- Opens TCP listeners on all configured addresses and port ranges (`target.go:openListeners`).
- Loads the PEM certificate and key from disk at startup (`target.go:newTLSConfig`).
- Logs the listener counts at startup via slog (`target.go:runTarget`).

## Verification

- Run: `go test ./cmd/benchmarking/egress/internal/cmd/target/...`
