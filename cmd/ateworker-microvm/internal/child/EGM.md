# child

## Overview
Supervises child processes (virtualization daemons such as cloud-hypervisor and virtiofsd) that the microVM worker launches.

## Interface
- `child.go:Start(cmd *exec.Cmd) (*Process, error)` starts and registers a supervised child.
- `child.go:Process` carries the running child's state and teardown.

## Invariants
- ["None"]

## Side Effects
- Starts and signals OS child processes via `exec.Cmd`.

## Verification
- Run: `go test ./cmd/ateworker-microvm/internal/child/...`
