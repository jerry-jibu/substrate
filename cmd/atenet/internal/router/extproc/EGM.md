# extproc

## Overview

The external processing (ext_proc) gRPC server the atenet router serves to
its dataplane gateways. This package is the multiplexer and nothing else: it
terminates the ext_proc stream, works out which direction (ingress or
egress) a request arrived on, dispatches to the `Handler` registered for
that direction, and records latency and outcome. Routing and authentication
policy lives in the sibling `ingress` and `egress` packages, which apply
opposite trust models and are deliberately kept apart.

## Interface

- `extproc.go:Server`, `extproc.go:NewServer` - the ext_proc mux over a `Handlers` map; serving a subset of directions is how `--mode` restricts an instance.
- `extproc.go:(*Server).Process` - the `ExternalProcessor_ProcessServer` stream loop.
- `extproc.go:(*Server).NewGRPCServer` - builds the gRPC server with the ext_proc service registered; the caller (Run/drain sequence) owns its lifecycle.
- `extproc.go:(*Server).Queries`, `(*Server).Recorder` - recent requests for the /statusz page and tests.
- `handler.go:Handler` - per-direction policy interface (`Direction`, `HandleRequestHeaders`).
- `handler.go:Result` - allowed-request outcome: header mutation, target, template attributes, resume outcome, dynamic metadata.
- `dispatch.go:Direction`, `dispatch.go:IsEgressFilterChain` - direction type and the egress filter chain names (`egress`, `egress_tls_mitm`, `egress_cleartext`).
- `errors.go:ReqError`, `NewReqError`, `WrapReqError`, `ImmediateResponse` - denial errors carrying a status code and client-safe body.
- `metadata.go:RequestMetadata`, `NewRequestMetadata` - normalized request headers and Envoy attributes (`:authority`).
- `record.go:QueryRecorder`, `RecordedQuery` - ring buffer of recent queries.
- `metrics.go:NewRouteDurationHistogram` - the route-duration histogram shared with handlers.
- `attributes.go` - Envoy attribute and format-string constants (`xds.filter_chain_name`, egress dynamic metadata keys).

## Invariants

- Dispatch is by filter chain, never by `:method`: on egress actor identity comes from a client certificate Envoy validated, while on ingress every header is unauthenticated client input; keying on `:method` would let any client select the egress handler and use its denials as an actor-existence oracle (`dispatch.go:directionOf`).
- An unrecognized or absent attribute means ingress, the fail-safe direction: a misrouted egress request fails on the missing routing header, whereas the reverse would leak control-plane state (`dispatch.go:directionOf`).
- The egress filter chain names must match the chain names in `manifests/ate-install/atenet-egress.yaml` (`dispatch.go` filter chain constants).
- A handler error denies the request: a `*ReqError` carries the status code and client-safe body, anything else becomes a 500 (`handler.go:Handler`).
- `Result` is read even when an error is returned, so handlers fill in metric attributes they learned (`handler.go:Result`).
- A direction absent from the `Handlers` map is refused at the mux (`extproc.go:Handlers`).
- Template atespace/name on `Result` are the low-cardinality route-duration attributes; empty when unresolved (`handler.go:Result`).

## Side Effects

- Serves the ext_proc gRPC stream to the dataplane gateway; registers OTel stats handler and records route-duration metrics per request.
- Keeps an in-memory ring buffer (100 entries) of recent queries for /statusz.

## Verification

- Run: `go test ./cmd/atenet/internal/router/extproc/...`
