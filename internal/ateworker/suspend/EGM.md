# suspend

## Overview

The upward half of a suspend: asks the control plane, through the node-local atelet, to suspend an actor the worker hosts. The suspend the control plane decides to run travels back down the ordinary path as a Checkpoint into the worker that asked, so a caller must hold no lock that Checkpoint needs while a request is in flight.

## Interface

- `suspend.go:Config` - `SocketPath`, `CredentialBundlePath`, `TrustBundlePath`, `AteletSPIFFEID`.
- `suspend.go:NewRequester(cfg) (*Requester, error)` - loads worker credentials once at startup.
- `suspend.go:Actor` - `Atespace`, `Name`, `UID`; the UID pins the request to the hosted incarnation.
- `suspend.go:Requester.RequestSuspend(ctx, actor) error` - asks that the actor be suspended and reports the control plane's answer.

## Invariants

- The retry budget is short on purpose (6 attempts, 500ms initial, factor 2, jitter 0.2, about fifteen seconds): an idleness signal decays, and suspend cancels in-flight requests rather than draining them (`suspend.go:retryBackoff`).
- Only Unavailable, Aborted, and ResourceExhausted are retried; refusals (NotFound, FailedPrecondition) and bugs (PermissionDenied, Unauthenticated, InvalidArgument) are answers, and DeadlineExceeded is not retried because it means the suspend itself is slow (`suspend.go:retryable`).
- One attempt is bounded by a 5-minute `requestTimeout` because the control plane answers only once the suspend it decided has finished, including snapshot upload (`suspend.go:requestTimeout`).
- The actor UID pins the request so it cannot outlive its actor and suspend a recreation of the same name (`suspend.go:Actor`).

## Side Effects

- Dials atelet over mTLS per request, picking up rotated credentials and re-verifying atelet each time (`suspend.go:Requester.RequestSuspend`).
- Calls `ateletpb.WorkerSupportClient.RequestActorSuspend` (`suspend.go:requestOnce`).
- Emits warning logs while retrying (`suspend.go:requestWithRetry`).

## Verification

- Run: `go test ./internal/ateworker/suspend/...`
