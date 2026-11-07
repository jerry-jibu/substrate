# metrics

## Overview

Mirrors the Prometheus surface emitted by the Python locust workers (`benchmarking/locust/common/metrics.py`): same metric names and labels, so dashboards keep working when the load source is a boomer-Go worker. Also bridges boomer stats (ZMQ to the locust master) and serves `/metrics`.

## Interface

- `prom.go:RecordSuccess(method, name, userClass, latency, responseBytes)` / `prom.go:RecordFailure(method, name, userClass, latency, errMsg)` - report to both boomer and Prometheus.
- `prom.go:RecordPromSuccess` / `prom.go:RecordPromFailure` / `prom.go:RecordServerLatency(method, name, userClass, latency)` - Prometheus only.
- `prom.go:UpdateUsers(userClass, delta)` - shifts the active-users gauge.
- `prom.go:Serve(ctx, addr) error` - runs the `/metrics` HTTP server until ctx is cancelled.

## Invariants

- Metric names and labels match the Python surface: `locust_requests_total{method,name,status,user_class}`, `locust_request_duration_milliseconds` (buckets 1..2^15 ms), `locust_server_duration_milliseconds` (buckets 0.1ms..~52s, no status label), `locust_users{user_class}` (`prom.go` var block).
- Use `RecordProm*` (not `RecordSuccess`/`RecordFailure`) once `boomer.Run` has returned: boomer may have stopped draining its stats channel and a send blocks when the buffer fills (`prom.go:RecordPromSuccess`).
- `Serve` shuts down gracefully within 5s of ctx cancellation and treats `http.ErrServerClosed` as success (`prom.go:Serve`).

## Side Effects

- Registers four Prometheus collectors at init (`prom.go:init`).
- Listens on a TCP address and serves `promhttp` (`prom.go:Serve`).
- Sends samples to the locust master via boomer (`prom.go:RecordSuccess`, `prom.go:RecordFailure`).

## Verification

- Run: `go test ./internal/benchmarking/boomer/metrics/...`
