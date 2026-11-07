# pid1

## Overview

Linux-only minimal init for the worker containers. When the process is PID 1, it re-executes the binary as its only child, reaps every orphan with `wait4(-1)` (which would otherwise race `os/exec` for subprocess exit statuses), forwards termination signals, and exits with the child's status.

## Interface

- `pid1.go:RunIfPID1()` - runs the init and exits when the process is PID 1; otherwise returns. Call it first thing in main.

## Invariants

- Only acts when `os.Getpid() == 1` (`pid1.go:RunIfPID1`).
- Sets `PR_SET_CHILD_SUBREAPER` before starting the child, and subscribes to signals before the child starts so no exit goes unnoticed (`pid1.go:supervise`).
- Reaps after every signal, not only SIGCHLD, because `os/signal` drops signals when the channel is full (`pid1.go:supervise`).
- Forwarded signals: SIGTERM, SIGINT, SIGHUP, SIGQUIT, SIGUSR1, SIGUSR2 (`pid1.go:forwarded`).
- Exit code is shell-style: 128 + signal for a signaled child, else its exit status (`pid1.go:exitCode`).

## Side Effects

- Re-executes the worker binary as a child process (`os.StartProcess` in `pid1.go:supervise`).
- Reaps all children with `unix.Wait4(-1, WNOHANG)` (`pid1.go:reap`).
- The init path ends the process via `os.Exit` with the child's status (`pid1.go:RunIfPID1`).

## Verification

- Run: `go test ./internal/ateworker/pid1/...`
