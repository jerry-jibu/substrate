# objectstore-plugin

## Overview
Serves the object-store plugin API (GCS or S3) over a Unix socket. Runs as a sidecar in two modes: `node` next to atelet for snapshot transfers and sandbox assets, and `control` next to ate-api-server, optionally also serving the credential exchange that trades actor JWTs for per-actor-scoped storage tokens. A third mode, `healthcheck`, is the sidecar's startup probe.

## Interface
- `main.go:main` - dispatches on `os.Args[1]`: `node`, `control`, or `healthcheck` (`main.go:usage`).
- Flags (`main.go:main`): `--socket` (required), `--root` (node; the only tree snapshot/asset files may touch), `--upload-memory-budget`, `--asset-staging-dir`, `--asset-staging-capacity`, `--timeout`, `--exchange-address` / `--exchange-ca-file` / `--exchange-server-name` (node, GCS), `--exchange-listen` / `--exchange-cred-bundle` / `--actor-jwt-issuer` / `--actor-jwt-issuer-ca-file` / `--transfer-service-account` / `--ateapi-address` / `--ateapi-ca-file` / `--ateapi-server-name` / `--ateapi-client-cred-bundle` (control), `--preview`.
- `main.go:healthcheck` - dials the plugin socket and waits up to `--timeout` for it to report serving.
- `exchange.go:serveExchange` / `exchange.go:newExchangeService` - control mode's TLS credential exchange; `exchange.go:exchangeTransportCredentials` - serving certificate re-read on rotation.
- gRPC services registered: `objectstorev1.RegisterNodeProviderServer`, `RegisterAssetProviderServer` (node), `RegisterControlProviderServer` (control), plus grpc health.

## Invariants
- Backend is chosen by `ATE_STORAGE_BACKEND` ("s3", or GCS by default) with ambient credentials (`main.go` package doc); `main.go:assetScheme` serves `gs` or `s3` URIs to match.
- On GCS every transfer runs on a token scoped to its actor from the exchange; S3 has no per-actor credential and uses the pod's own (`main.go:main` node branch).
- `exchange.go:newExchangeService` requires `--actor-jwt-issuer`, `--transfer-service-account`, and `--actor-jwt-issuer-ca-file` on GCS; S3 answers UNIMPLEMENTED.
- The exchange serves TLS 1.3 without client certificates: the JWT is the credential (`exchange.go:exchangeTransportCredentials`).
- `--asset-staging-dir` must not be below `--root` or readable by the caller (flag description); `main.go:parseCapacity` accepts only positive whole-byte Kubernetes quantities.
- On SIGTERM the socket server drains in-flight transfers and the exchange drains at the same time, not after (`main.go:main`).

## Side Effects
- Listens on a Unix socket (`pluginsocket.Listen`), and on TCP for the exchange in control mode.
- Reads and writes snapshot and sandbox-asset files under `--root`; creates `--asset-staging-dir` with mode 0700.
- Impersonates `--transfer-service-account` and downscopes tokens per actor (via `gcsissuer`); dials ate-api-server over loopback mTLS for actor lookups.
- Exits the process if the exchange's `Serve` fails, so the pod restarts rather than serving without an exchange (`exchange.go:serveExchange`).

## Verification
- Run: `go test ./cmd/objectstore-plugin/...`
