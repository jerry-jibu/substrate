# ateapi

## Overview
The ate-api-server binary. `main.go` wires the gRPC server: authentication (`apiauthn`, `ateletauth`), authorization (`authz`), the Control service (`controlapi`), the Worker service (`workerservice`), PostgreSQL persistence (`store/atepg`), actor JWT and certificate authorities, OTel instrumentation, and the metrics and health probe servers.

## Interface
- `main.go:main` performs the boot sequence inline (config load, flag/env setup, gRPC server assembly); `main.go:maxRPCDeadline` caps every served RPC at 10 minutes.
- Flags in `main.go` define the deployment surface: `--config-file`, `--grpc-listen-addr`, `--metrics-listen-addr`, `--health-listen-addr`, `--grpc-server-cred-bundle`, `--postgres-*` (read-write and owner connection strings, roles, schema, pool size), `--authz-bootstrap-owners`, `--controller-spiffe-id`, `--egress-gateway-spiffe-id`, `--ingress-router-spiffe-id`, `--api-server-sidecar-spiffe-id`, `--actor-id-jwt-pool`, `--actor-jwt-issuer`, `--default-egress-gateway-address`.

## Invariants
- `--authz-bootstrap-owners` requires at least one ID; removal of an ID revokes its access on restart (`main.go` flag doc).
- `--actor-jwt-issuer` must be an https URL with no query or fragment (`main.go` flag doc).
- Authorization is always enforced against the embedded OpenFGA model; SPIFFE IDs hold system roles when calling over mTLS (`main.go` SPIFFE flag docs).

## Side Effects
- Listens on the gRPC, metrics, and health probe ports (`main.go` flags).
- Connects to PostgreSQL with separate read-write and owner credentials (`main.go` postgres flags).
- Signs actor JWTs and serves the actor ID JWKS via the configured issuer.

## Verification
- Run: `go test ./cmd/ateapi/...`
