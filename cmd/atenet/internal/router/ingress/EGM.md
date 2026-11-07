# ingress

## Overview

The ext_proc handler for traffic arriving at the ingress gateway: resolves
the actor a request is addressed to, resumes it through the control plane
(parking the request while the worker pool is saturated), and points the
dataplane at the worker that ends up hosting it. Everything reaching this
handler is unauthenticated client input; the opposite trust model belongs to
the sibling `egress` package.

## Interface

- `ingress.go:Handler`, `ingress.go:New` - the `extproc.Handler` for `DirectionIngress`; `New` wires the `ActorResumer` and parking lot.
- `ingress.go:(*Handler).HandleRequestHeaders` - parses the target actor, resumes it, returns the routing result (original-dst metadata, worker SPIFFE ID, target port).
- `ingress.go:(*Handler).ParkingStatus` - parking lot snapshot for /statusz.
- `ingress.go` metadata keys - `OriginalDstMetadataKey`/`OriginalDstAddressKey`/`OriginalDstPortKey` (read by the xds ORIGINAL_DST cluster) and `WorkerSPIFFEIDKey`.
- `resumer.go:ActorResumer`, `NewActorResumer`, `ResumeOutcome` - deduplicated actor resumption with per-actor singleflight flights.
- `parking.go:ParkedRequestConfig`, `DefaultParkedRequestConfig`, `ParkingStatus` - parked-request knobs (budget, max, retry cadence) and their defaults.
- `parking.go:parkingLot` (unexported) - bounded admission for parked requests.
- `metrics.go:ParkingMetrics`, `NewParkingMetrics` - parking wait duration histogram by outcome.
- `errors.go:mapResumeError`, `actorNotFoundErr`, `parkingFullErr` (unexported) - map resume failures to client-safe HTTP statuses.

## Invariants

- The handler treats every header as unauthenticated client input; duplicate or comma-carrying routing headers are rejected rather than disambiguated (`ingress_test.go` behavior enforced by `HandleRequestHeaders` parsing through `atenet.ParseTargetActor`).
- CONNECT port selection uses the retained outer authority from filter state; the inner `:authority` after CONNECT termination is not trusted for the port (`ingress.go:HandleRequestHeaders`).
- Concurrent resumes of one actor share a single flight (`flights` map); the first caller is `ResumeOutcomeTriggered`, joiners are `ResumeOutcomeJoined` (`resumer.go:ActorResumer`, `flight.go:resumeActorFlight`).
- A caller that cancels does not abort the shared flight (`flight.go` publish/`callerResult` split).
- Parking has no separate on/off switch: `Max` <= 0 disables it, making capacity errors fail fast; only concurrent-update conflicts retry, bounded by `failFastResumeBudget` (15s) (`parking.go:Enabled`, `resumer.go:failFastResumeBudget`).
- The retry backoff deliberately sets no Cap and a high Steps count so the parking budget, not the step count, bounds the wait (`resumer.go:resumeBackoff`).
- `DefaultParkedRequestMax` (1024) is sized against the ext_proc cluster's circuit breaker; each parked request holds one ext_proc stream for its entire wait (`parking.go:DefaultParkedRequestMax`).
- Budget exhaustion still maps the underlying gRPC status faithfully (e.g. 503 with the capacity message) while being its own parking metric outcome (`resumer.go:budgetExhaustedError`).

## Side Effects

- Calls `ateapi.ControlClient` (ResumeActor) over the network; emits OTel spans (trace context extracted from the request headers), route-duration and parking metrics.
- Holds in-memory parking-lot and flight state; parked requests hold their ext_proc streams open.

## Verification

- Run: `go test ./cmd/atenet/internal/router/ingress/...`
