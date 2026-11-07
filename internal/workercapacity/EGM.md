# workercapacity

## Overview

Names the files atecontroller projects into the worker container from the
downward API, in millicores and bytes.

## Interface

- Constants: `workercapacity.go:MountPath` (`/run/worker-capacity`), `workercapacity.go:CPULimitFile` (`cpu_millicores`), `workercapacity.go:MemoryLimitFile` (`memory_bytes`).
- Subdirectories: none.

## Invariants

- File contents are millicores and bytes respectively; both sides of the projection must keep the names in step.

## Side Effects

- ["None"]

## Verification

- Run: `go test ./internal/workercapacity/...`
