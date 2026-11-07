# actor

## Overview

Workload of the egress scale test. The `actor` cobra subcommand holds one
HTTP tunnel per target endpoint open through the egress gateway and counts
what happened. It serves its control routes on port 80 (the only inbound
port of a sandbox) and is driven entirely over HTTP: a driver POSTs
`/start`, polls `GET /stats`, cuts windows with `POST /collect`, and ends
the run with `POST /stop`.

## Interface

- `actor.go:Command` - cobra command `actor` with `--listen` (default `:80`); serves routes until context cancel.
- `api.go:ReadyzRoute`, `api.go:StartRoute`, `api.go:StatsRoute`, `api.go:CollectRoute`, `api.go:StopRoute` - control routes `/readyz`, `/start`, `/stats`, `/collect`, `/stop`.
- `api.go:ProtocolHeader`/`api.go:ProtocolVersion` - every answer carries `Benchmark-Egress-Protocol: 2`; drivers must require 2.
- `api.go:StartRequest` - JSON body of `POST /start` (`endpoints`, `requestTimeoutMs`, `intervalMs`, `offsetMs`, `scheme`); `StartRequest.Validate` enforces `1..api.go:MaxEndpoints` (1024), non-negative durations, offset within one period.
- `api.go:WindowParam` - `window` query parameter of `POST /collect`; cut ids start at 1.
- `actor.go:server.handler` - mux serving all routes; every answer sets the protocol header.
- `actor.go:loopRun` - one started loop: per-endpoint goroutines, counters under `loopRun.mu`, distributions in the recorder.
- `actor.go:loopRun.collect` - cuts a window; answers the last cut's id again, rejects out-of-sequence ids with 409 (`actor.go:sequenceError`).
- `actor.go:loopRun.stop` - cancels the loop, waits for all goroutines, returns final Stats; the final Stats stays answerable by repeated `/stop`.
- `actor.go:classify` (in classify.go) - maps failures to low-cardinality classes (`dns: not found`, `tls: unknown authority`, `timeout`, `connection reset`, ...); `""` means success.
- `endpoints.go:EndpointHost`, `endpoints.go:EndpointURL`, `endpoints.go:ServiceName` - endpoint `i` is `egress-target-<i>.egress-tests-targets.svc.cluster.local`; `endpoints.go:EndpointHostPattern` is `*.egress-tests-targets.svc.cluster.local`.
- `hist.go:Bounds` - 157 bucket bounds, 20 per decade from 1 microsecond to 63.1 seconds, parsed from decimal strings so every build holds identical values.
- `hist.go:Point`, `hist.go:Point.Merge`, `hist.go:Point.Quantile`, `hist.go:Point.Mean` - JSON-mirror of an OTel histogram data point; `Merge` refuses mismatched bounds.
- `hist.go:Distribution` (`Opening`, `Latency`, `DNS`, `TLS`) - the four distributions, keys of `Stats.Distributions`.
- `recorder.go:newRecorder`, `recorder.go:recorder.record`/`collect`/`shutdown` - OTel histograms on a private `ManualReader` in delta temporality; nothing is exported.
- `stats.go:Stats`, `stats.go:Endpoint` - counters cumulative since `/start`; per-endpoint counts and distributions served only on request.
- Subdirectories: none.

## Invariants

- The loop starts on `POST /start`, not at boot, so the golden snapshot taken after template warmup does not freeze a running loop (`actor.go:Command` doc comment).
- At most one loop runs at a time; a second `/start` answers 409 (`actor.go:server.handleStart`).
- Per-endpoint counts and distributions are served only with `?endpoints=1`; the kept final Stats is never mutated (`actor.go:view`).
- `GET /stats` never consumes state; only `/collect` and `/stop` consume a window, each sample is served once (`api.go:CollectRoute` comment).
- Requests are HTTP/1.1 with keep-alive, one idle connection per endpoint, no idle timeout, redirects not followed (`actor.go:newTransport`, `actor.go:startLoop` `CheckRedirect`).
- `Stats.Rounds` is the least-visited endpoint's request count (`actor.go:record`, `actor.go:loopRun.rounds`).
- A request failing inside the TLS handshake is classed `tls handshake: ...` when the handshake started but did not complete and the class is not already `tls: ...` (`actor.go:record`).
- Reads are bounded by `actor.go:maxBodyBytes` (1 MiB) for both `/start` bodies and endpoint responses.

## Side Effects

- Serves an HTTP server on `--listen` (default `:80`) (`actor.go:runActor`).
- Opens outbound HTTP connections to the endpoint URLs, DNS lookups and TLS handshakes included (`actor.go:loopRun.request`, `actor.go:requestTrace`).
- Reads `SSL_CERT_FILE`-extended system roots for HTTPS verification unless overridden (`actor.go:server.rootCAs` comment).

## Verification

- Run: `go test ./cmd/benchmarking/egress/internal/cmd/actor/...`
