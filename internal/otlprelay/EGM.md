# otlprelay

## Overview

Carries worker OTLP telemetry to the collector over a unix socket served by
atelet, so a worker pod needs no network path of its own to export spans,
metrics, and log records. Payloads are forwarded verbatim; the request's
metadata is replaced with headers atelet resolves from its own
`OTEL_EXPORTER_OTLP_HEADERS`.

## Interface

- `relay.go:NewServer` - builds the atelet-side relay: an OTLP receiver on `sockPath` forwarding to the collector named by the standard OTLP env vars; returns `(nil, nil)` when the socket path or endpoint is empty (relay off).
- `relay.go:Server.Serve` - listens on the unix socket (mode `relay.go:socketMode` 0600), removing a stale socket first; `relay.go:Server.Stop` drains, closes upstream, removes the socket.
- `relay.go:traceRelay`/`metricRelay`/`logRelay` - `Export` forwarders for the three OTLP services; payloads pass through unchanged.
- `relay.go:sourceGate` - allowlist `relay.go:workerServices` (`ateworker-gvisor`, `ateworker-microvm` by `service.name`); rejects others with PermissionDenied, logging once per distinct name through an LRU of 256 (`relay.go:check`).
- `relay.go:upstreamContext`/`parseHeaders`/`upstreamHeaders` - per-signal header resolution from atelet's environment; client metadata is dropped, never forwarded.
- `relay.go:upstreamTarget`/`resolvePerSignal`/`normalizeEndpoint` - endpoint resolution; signal-specific endpoint vars must agree; `https` endpoints are rejected, not downgraded; default port `relay.go:otlpDefaultPort` (4317).
- `relay.go:upstreamCompression` - gzip or none; `relay.go:maxRecvMsgSize` (16 MiB) bounds one Export payload.
- `client.go:Dial` - worker half: a gRPC connection over `unix://<sockPath>`; returns `(nil, nil)` when the path is empty or absent so the caller falls back to the OTLP endpoint.
- `relay.go:validateSocketPath` - rejects relative socket paths.

## Invariants

- The relay forwards OTLP request messages verbatim and never injects its own spans, so worker resources stay attributed to the worker (`relay.go` package comment, `relay.go:traceRelay.Export`).
- Client-supplied metadata is dropped; upstream headers come only from atelet's own environment because they are credentials (`relay.go:upstreamContext` doc).
- The source allowlist is a protocol contract, not a security boundary: `service.name` is client-provided (`relay.go:workerServices` comment).
- A batch whose resource fails the gate is refused whole; partial success would lose telemetry silently (`relay.go:traceRelay.Export`).
- The relay dial is plaintext by design (unix socket cannot leave the node; access is file permissions) (`client.go:Dial` doc, `relay.go:NewServer` insecure credentials).
- Upstream is plaintext: an `https` OTLP endpoint is an error, not a downgrade (`relay.go:normalizeEndpoint`).
- The upstream gRPC client is lazy; atelet startup does not depend on the collector (`relay.go:NewServer` comment).
- Header values never reach logs; only sorted header names are logged (`relay.go:headerNames`, `relay.go:parseHeaders` error messages omit values).

## Side Effects

- Creates the socket directory, removes a stale socket, listens on the unix socket, and chmods it 0600 (`relay.go:Server.Serve`).
- Opens a gRPC connection to the collector (insecure credentials, optional gzip) (`relay.go:NewServer`).
- Reads `OTEL_EXPORTER_OTLP_*` environment variables (endpoints, compression, headers, with signal-specific overrides) (`relay.go:upstreamTarget`, `relay.go:upstreamCompression`, `relay.go:upstreamHeaders`).

## Verification

- Run: `go test ./internal/otlprelay/...`
