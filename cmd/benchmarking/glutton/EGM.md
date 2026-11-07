# glutton

## Overview

A small benchmarking workload: a gRPC (or HTTP, per `--mode`) server for
consuming RAM, disk, and file descriptors, and for gossiping with other
glutton instances. The service API is defined in
`internal/proto/glutton/glutton.proto`; the implementation lives in
`internal/benchmarking/glutton`.

## Interface

- `main.go:main` - entry point. Flags: `--grpc-listen-addr` (default `:8080`, name kept for back-compat), `--metrics-listen-addr` (default `:9090`), `--data-dir` (required), `--mode` (`grpc`|`http`, default grpc), `--sync-writes`, `--readyz-require-writable-data-dir`, `--version`.
- `main.go:main` wires `glutton.New(*dataDir, opts...)`, `glutton.Handler(*mode, svc)`, and `glutton.NewServer(handler).Serve(lis)` from `internal/benchmarking/glutton`.
- Subdirectories: none.

## Invariants

- `--data-dir` is required; the binary exits 2 without it (`main.go:main`).
- Tracing and metrics providers are shut down via `serverboot.ShutdownProvider` defers (`main.go:main`).

## Side Effects

- Creates `--data-dir` with mode 0700 (`main.go:main` `os.MkdirAll`).
- Listens on the gRPC address and starts the Prometheus metrics server (`main.go:main`).
- With `--sync-writes`, fsyncs each WriteDisk file before returning (`main.go:--sync-writes` flag via `glutton.WithSyncWrites`).
- Initializes OTel tracing (`serverboot.InitTracing`) and metrics (`serverboot.InitMetrics`).

## Verification

- Run: `go test ./cmd/benchmarking/glutton/...`
