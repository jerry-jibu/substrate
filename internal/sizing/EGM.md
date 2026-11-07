# sizing

## Overview

Right-sizes a sandbox to the actor's declared resource limits. The limits
arrive on the worker RPCs (RunWorkload / RestoreWorkload). gVisor applies
them to the sandbox container's OCI spec; microVM uses them only to size
the VM itself.

## Interface

- `sizing.go:SandboxSize` - `MilliCPU` and `MemoryBytes`; zero means unset.
- `sizing.go:FromLimits` - builds one from `ateworkerpb.Resources`; negatives clamp to zero.
- `sizing.go:SandboxSize.VCPUs` - whole vCPUs for the micro-VM, rounding up, minimum 1 when set; 0 when unset.
- `sizing.go:SandboxSize.ApplyToOCISpec` - writes CPU quota (period `sizing.go:cpuQuotaPeriodMicros` 100000) and memory limit into `linux.resources`; unset fields leave the spec untouched; called only by ateworker-gvisor.

## Invariants

- `ApplyToOCISpec` preserves existing spec values for unset fields and only the gVisor runtime applies it; a microVM container's own cgroup limit comes from its declared container resources instead (`sizing.go` package comment, `sizing.go:ApplyToOCISpec`).

## Side Effects

- ["None"] (mutates only the in-memory spec passed to it)

## Verification

- Run: `go test ./internal/sizing/...`
