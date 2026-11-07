# egress

## Overview

Demo and e2e vehicle for per-Actor egress policy. `main.go` is an HTTP
service that fetches a caller-supplied URL and returns the upstream
response, plus endpoints exercising gRPC, WebSocket, HTTP CONNECT, delayed
headers/bodies, and connection multiplexing, so an e2e can assert what
survives the egress path. `test-egress.sh` runs positive and negative
identity checks against a deployed demo install.

## Interface

- `main` (`main.go:158`): listens on `:80`, JSON in and out.
- Endpoints (`main.go:169`-`:238`): `/readyz`; `/` fetches a URL (POST
  `fetchRequest{url, rootCA, http1}`, `main.go:64`); `/post` sends a body
  after a 15s delay (`handlePost`, `main.go:232`); `/slow-headers` sends
  headers without the terminating blank line for 5s (`handleSlowHeaders`,
  `main.go:233`); `/mux` opens 10 concurrent requests that send headers
  first and bodies after all headers are out (`handleMux`, `main.go:461`);
  `/websocket` (`handleWebSocket`, `main.go:641`); `/grpc` runs unary,
  server-stream, and bidirectional echo RPCs against
  `internal/proto/grpcechopb` (`handleGRPC`, `main.go:756`); `/connect`
  sends one HTTP CONNECT to a forward proxy (`handleConnect`,
  `main.go:892`).
- Validators: `validateURL` (`main.go:991`), `validateTarget`
  (`main.go:1011`), `validateWebSocketRequest` (`main.go:670`).
- `test-egress.sh`: verifies a running actor's plain-HTTP egress is tunneled
  and authorized (200), and that a pod identity without an actor certificate
  is refused at the mTLS handshake; `--cleanup` removes what it created
  (`test-egress.sh:17`-`:32`).
- Install templates: `egress.yaml.tmpl` / `egress-microvm.yaml.tmpl` and
  their `-template` ActorTemplate files.

## Invariants

- Requests are bounded: 64 KiB request bodies, 1 MiB response bodies, 15s
  request timeout, at most 16 WebSocket messages of 64 KiB
  (`main.go:50`-`:61`).
- The gRPC connection is dialed per request and closed with it, because an
  HTTP/2 connection opened before a snapshot does not survive restore
  (`main.go:777`-`:787`).
- `echoBidi` sends one message at a time, waiting for each response, and
  half-closes before draining, so one-directional paths fail the endpoint
  (`main.go:833`-`:876`).
- Request-scoped TLS starts from a cloned default transport with no proxy
  and minimum TLS 1.2, so it cannot inherit an insecure callback
  (`fetchClient`, `main.go:606`-`:639`).

## Side Effects

- Makes outbound HTTP, TLS, WebSocket, gRPC, and CONNECT connections to
  caller-supplied targets (`main.go:193`-`:231`, `:782`, `:945`).
- Propagates an inbound `traceparent` header to the outbound request
  (`main.go:198`-`:200`).
- `test-egress.sh` creates and deletes cluster resources (actors, probe
  pod) via `kubectl` and `kubectl-ate` (`test-egress.sh:26`-`:32`).

## Verification

- Run: `go test ./demos/...`
