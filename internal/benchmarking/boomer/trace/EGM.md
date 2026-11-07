# trace

## Overview

Wires the OTLP tracer and W3C TraceContext propagator used by the boomer load-test workers: same shape as `internal/serverboot.InitTracing` without the metrics/HTTP surface workers do not need. Includes a sampler whose probability can be swapped at runtime, mirroring `common/trace.py:UpdatableSampler` on the Python side.

## Interface

- `trace.go:UpdatableSampler` with `NewUpdatableSampler(initialProbability)`, `UpdateProbability(p)`, `ShouldSample`, `Description` - implements `sdktrace.Sampler`.
- `trace.go:Init(ctx, serviceName, sampler) (*sdktrace.TracerProvider, error)` - registers the global TracerProvider with the given sampler, an OTLP gRPC batch exporter (insecure, honors `OTEL_EXPORTER_OTLP_ENDPOINT`), and the W3C propagator.

## Invariants

- `serviceName` is required (`trace.go:Init`).
- Each `UpdateProbability` installs a fresh `ParentBased(TraceIDRatioBased(p))` sampler atomically; the OTel TracerProvider accepts only one sampler at construction, hence the stable wrapper (`trace.go:UpdatableSampler.UpdateProbability`).
- `Entry.UserClass` must equal the boomer task name for the master's spawn messages to match; that contract is declared by the sibling userclass package (`userclass/registry.go:Entry`).
- Partial-resource and schema-URL-conflict errors are tolerated with a warning so a missing `OTEL_*` env var does not fail the worker (`trace.go:newResource`).

## Side Effects

- Sets the global OTel TracerProvider, propagator, and error handler (`trace.go:Init`).
- Exports spans over OTLP gRPC; SDK errors (export failures, queue drops) are routed into `slog` so they land in boomer-worker's stdout (`trace.go:Init`).
- Generates a UUID service instance ID per worker (`trace.go:newResource`).

## Verification

- Run: `go test ./internal/benchmarking/boomer/trace/...`
