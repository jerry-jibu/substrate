# pidfd

## Overview

Watches for processes to exit without holding a thread per process, via
Linux `pidfd_open`, so a worker can watch thousands of them. Needs Linux
5.3+; before 5.10 (`PIDFD_NONBLOCK`) each watch falls back to a blocking
poll on its own thread (`doc.go`).

## Interface

- `pidfd.go:Exited` - pins `pid` with `PidfdOpen` and returns a channel closed at exit; ESRCH yields an already-closed channel; EINVAL (pre-5.10) retries without `PIDFD_NONBLOCK`.
- `pidfd.go:ExitedFD` - same for a process already pinned by a caller-held pidfd; watches a `F_DUPFD_CLOEXEC` duplicate so the caller keeps its fd.
- `pidfd.go:watch` - closes `done` when the pidfd turns readable; a nonblocking fd joins the runtime netpoller, otherwise falls back to a blocking 1s poll loop (`pidfd.go:readable`).
- Off Linux both functions return `errors.ErrUnsupported` (`pidfd_other.go`).

## Invariants

- The pid is pinned at call time, so a later reuse of the PID number is never mistaken for the process (`pidfd.go:Exited` doc).
- The watch verifies readability of the fd itself rather than trusting a netpoller wakeup, because `Read` drops a readiness edge that arrived before it started (`pidfd.go:watch` comment).
- The blocking-poll fallback never reports an exit that did not happen, at the cost of holding a thread (`pidfd.go:watch` comment).
- `ExitedFD` duplicates rather than adopts the caller's pidfd; the duplicate shares status flags, so the caller's fd becomes nonblocking too (`pidfd.go:ExitedFD` comment).

## Side Effects

- Opens pidfds and duplicates file descriptors (`pidfd.go:Exited`, `pidfd.go:ExitedFD`); each watch owns and closes its fd (`pidfd.go:watch`).

## Verification

- Run: `go test ./internal/pidfd/...`
