# procstatus

## Overview

Holds the JSON shape the e2e testserver's `procstatus` subcommand prints: the kernel's view of processes on a worker, read from `/proc`. Shared between the testserver fixture and the worker security e2e tests that compare it.

## Interface

- `procstatus.go:Process` - one process's `/proc/<pid>/status` reduced to the compared fields: `PID`, `PPID`, `Name`, capability masks (`CapPrm`, `CapEff`, `CapBnd`, `CapAmb`), `NoNewPrivs`, `Seccomp`, and `Threads`.
- `procstatus.go:Thread` - one thread from `/proc/<pid>/task/<tid>/status`: `TID`, `Name` (the thread's comm, e.g. cloud-hypervisor's "vmm", "vcpu0"), `Seccomp`.

## Invariants

- Capability sets are raw 64-bit masks, not decoded names; seccomp on `Process` is the main thread's mode while `Threads` carries every thread's (`procstatus.go:Process` doc).

## Side Effects

- ["None"] (pure data types; the reading happens in the testserver fixture).

## Verification

- Run: `go test ./internal/e2e/procstatus/...`
