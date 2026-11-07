# fake

## Overview

An httptest-backed stand-in for a glutton actor, for exercising the benchmark drivers without a sandbox. The `Data` slice is the source of truth for size, digest, and payload; each override field makes the actor lie about exactly one property so client-side verification can be tested.

## Interface

- `server.go:Server` - the stand-in. Fields: `Data`, `Digest`, `CorruptPayload`, `EmptyPayload`, `Status` (fail every route with an HTTP status), `ElapsedUs` (sets the `x-server-elapsed-us` header).
- `server.go:Server.Start(t) *httptest.Server` - starts with `t.Cleanup` teardown; `server.go:Server.HexDigest()`.
- Route constants re-exported from the real server: `server.go:WriteDiskRoute`, `ReadDiskRoute`, `WriteRAMRoute`, `ReadRAMRoute`, `BurnCPURoute`, `IngestRoute`, `PingRoute`, `UseCPURoute`, `RunScriptRoute`.
- Request recorders: `RecordedPaths`, `RecordedWriteSizes`, `RecordedReadModes`, `RecordedRAMWriteSizes`, `RecordedRAMWriteModes`, `RecordedRAMReadSizes`, `RecordedBurnMillis`, `RecordedIngestSizes`, `RecordedCPURequests`, `RecordedScripts` (`server.go` methods).

## Invariants

- Routes are re-exported from the real glutton server, so the stand-in cannot answer a path the actor does not (`server.go` route const block).
- Both disk routes report `len(Data)` and `sha256(Data)` unless overridden; `/readdisk` serves `Data` as payload; `EmptyPayload` takes precedence over `CorruptPayload` (`server.go:Server` doc, `server.go:reportedPayload`).
- `/ingest` digests the actual request payload (`server.go:serve` IngestRoute).
- `/runscript` does no work: `mirrorBlock` builds the result a zero-cost script would produce, one pass per node and one request per leaf (`server.go:mirrorBlock`).
- Recordings are guarded by a mutex and returned as copies (`server.go` recorder methods).

## Side Effects

- Binds an httptest listener and serves protobuf-over-HTTP (`server.go:Start`).

## Verification

- Run: `go test ./internal/benchmarking/glutton/fake/...`
