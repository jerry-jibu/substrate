# ateapi/internal/scheduling

## Overview

Decides which worker should host an actor (`scheduling.go` package doc).

## Interface

- `Scheduler`, `New`, `Option`, `WithIntn` (`scheduling.go`) - the placement API and its constructor; `WithIntn` overrides the random source for tests.
- `Schedule`, `Applies`, `HasRoom` (`scheduling.go`) - placement questions against the current fleet.
- `Constraints` (`scheduling.go`) - what a worker must satisfy: SandboxClass, RequiredRuntime, SnapshotRuntime, TemplateSelector/ActorSelector, RequiredNodes, Limits.
- `ErrNoCapacity` (`scheduling.go`) - no worker that meets the constraints has room.
- `WorkerSource` (`scheduling.go`) - supplies the whole fleet (satisfied by the workercache).

## Invariants

- `SandboxClass` is never relaxed: a snapshot's files are written by one class's worker and read back by the same, unlike `RequiredRuntime` (`scheduling.go` `Constraints`).
- `SnapshotRuntime` is never relaxed either: it picks the worker runtime the actor runs under and the overhead it is booked with (`scheduling.go` `Constraints`).
- `Schedule` uses the power of two choices: sample two candidates at random and return the less-loaded one, where load is the higher of actor-slot and compute-resource utilization (`scheduling.go` `Schedule`).
- `Applies` excludes capacity so an existing assignment does not make its Worker ineligible; `HasRoom` is independent of `Applies` (`scheduling.go` `Scheduler`).

## Side Effects

- ["None"]

## Verification

- Run: `go test ./cmd/ateapi/internal/scheduling/...`
