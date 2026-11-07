# pluginsocket

## Overview

Connects a server to its plugin sidecar over a Unix socket. Depends on no
plugin implementation, so a server that only calls a plugin does not link
the plugin's storage SDKs.

## Interface

- `pluginsocket.go:Listen` - removes any stale socket at `path` and listens on a fresh one chmod 0600 (owner-only); creates the parent dir 0700.
- `pluginsocket.go:Dial` - long-lived insecure gRPC client over `unix://<path>` with OTel stats handler, fast reconnect backoff (`pluginsocket.go:reconnectParams`, max delay 1s), and a bounded-wait unary interceptor.
- `pluginsocket.go:ReadyWait` - the ready wait (30s) servers pass to `Dial`; survives a sidecar restart including the first 10s of kubelet crash-loop backoff.
- `pluginsocket.go:boundedWaitForReady`/`awaitReady` - unary calls wait up to `readyWait` for connectivity.Ready, then fail with `codes.Unavailable`; a caller-set `grpc.WaitForReady`/FailFast option opts out of the bound.
- `pluginsocket.go:WaitReady` - blocks on the gRPC health check (`grpc.WaitForReady(true)`) until the plugin reports SERVING; bounding ctx turns a never-coming plugin into a startup error.
- `pluginsocket.go:CallError` - maps `codes.Unavailable` to `apierror.Unavailable` so the server's caller retries.

## Invariants

- The bound covers unary calls only; a streaming call gets plain fail-fast and needs its own stream interceptor for the same wait (`pluginsocket.go:Dial` doc).
- A call already on a ready connection is not bounded by `readyWait` (`pluginsocket.go:Dial` doc).
- The connection is unauthenticated; the socket is reachable only from inside the pod by its owner (`pluginsocket.go:Dial` doc, `pluginsocket.go:Listen` chmod).

## Side Effects

- Creates the socket directory, removes a stale socket, and listens on a 0600 unix socket (`pluginsocket.go:Listen`).
- Opens a gRPC connection to the plugin socket (`pluginsocket.go:Dial`) and issues gRPC health checks (`pluginsocket.go:WaitReady`).

## Verification

- Run: `go test ./internal/pluginsocket/...`
