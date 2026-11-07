# wakeupprobe

## Overview

Polls a container's HTTP wakeup endpoint from inside the worker to detect
the moment its server starts accepting connections with millisecond
latency: pre-bind, the kernel answers RST in microseconds, so a
sub-millisecond poll loop is nearly free, and once the socket is up the
next GET completes at veth-local latency.

## Interface

- `wakeupprobe.go:WaitAll` - blocks until every container with a `WakeupProbe` reports 200 (`wakeupprobe.go:Wait`); containers without one are skipped (no wakeup gate).
- `wakeupprobe.go:Wait` - polls until 200, ctx cancel, or the probe's `timeout_seconds`; failure names attempts and the last error.
- `wakeupprobe.go:URL` - builds `http://<actorIP>:<port><path>`; requires `httpGet`, a port in 1-65535, and a path (prefixed with `/` if missing).
- `wakeupprobe.go:DialFunc` - reaches the actor across its network namespace; nil dials from the caller's namespace.
- `wakeupprobe.go:HTTPClient` - keep-alive client tuned for single-endpoint probing (`wakeupprobe.go:newClient`); a var so tests substitute a transport.
- Constants: `wakeupprobe.go:RequestTimeout` (250ms), `wakeupprobe.go:PollInterval` (1ms), `wakeupprobe.go:maxIdleConnsHost` (1).
- Subdirectories: none.

## Invariants

- Only HTTP 200 passes the gate; other statuses are errors (`wakeupprobe.go:tryOnce`).
- `timeout_seconds` must be positive (`wakeupprobe.go:pollTimeout`).
- The probe loop sleeps between attempts rather than busy-looping; responses are drained so the keep-alive connection is reused (`wakeupprobe.go:Wait`, `wakeupprobe.go:tryOnce`).

## Side Effects

- Sends HTTP GETs to the actor's wakeup endpoint through the supplied dialer (`wakeupprobe.go:Wait`).

## Verification

- Run: `go test ./internal/wakeupprobe/...`
