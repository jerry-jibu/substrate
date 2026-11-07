# agentstats

## Overview
Converts the kata agent's cgroup statistics into the sample type the microVM worker reports.

## Interface
- `agentstats.go:FromCgroupStats(cs *agentpb.CgroupStats) Sample` converts agent-reported cgroup stats.
- `agentstats.go:Sample` carries the converted usage values.

## Invariants
- ["None"]

## Side Effects
- ["None"] (pure conversion; the gRPC read happens in the kata agent client).

## Verification
- Run: `go test ./cmd/ateworker-microvm/internal/agentstats/...`
