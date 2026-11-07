# dynconfig

## Overview

Fetches and holds the boomer worker's runtime-mutable settings: the locust flags an operator can change in the web UI, delivered over an HTTP side channel from the master's `/boomer-config` endpoint. A worker runs exactly one user class, which hands the Holder a typed Codec (defaults plus validation); this package stores the typed value, decodes each payload over the defaults, refuses a payload the class cannot run on, and serves reads via `Get`.

## Interface

- `dynconfig.go:Codec` - `Initial() any` and `Decode(payload []byte) (any, error)`; `dynconfig.go:Typed[T]` implements it over a knobs struct with `Defaults` and `Validate`.
- `dynconfig.go:Holder` / `dynconfig.go:Snapshot` - holds the class config and the worker's `Common` slice (`trace_probability`) swapped together atomically.
- `dynconfig.go:NewHolder(codec)` / `dynconfig.go:Static[T](cfg)` (tests) / `dynconfig.go:Holder.Apply(payload) (Snapshot, bool, error)`.
- `dynconfig.go:Get[T](h *Holder) T` - typed read; `dynconfig.go:Holder.Common()`.
- `dynconfig.go:Seconds` - JSON duration in seconds; `dynconfig.go:WaitTime` with `Validate`/`Draw`; `dynconfig.go:Uniform(lo, hi)`.
- `dynconfig.go:Lifecycle` with `ResumeMode` (`explicit`/`implicit`) and `LifecycleMode` (`sync`/`async`/`none`) and their constants.
- `dynconfig.go:Fetch(ctx, url)`; `dynconfig.go:StartPoll(ctx, url, holder, sampler, interval, fetchTimeout, onError)`; `dynconfig.go:SubscribeSpawn(url, holder, sampler, fetchTimeout, onError)`.

## Invariants

- Each payload stands on its own: a key absent or null in it is at the class's default, not the previous value; a body that is not a JSON object, an empty one included, is refused (`dynconfig.go:Codec.Decode` contract).
- A payload the class codec or `Common` refuses leaves the holder at the last good config (`dynconfig.go:Holder.Apply`).
- `trace_probability` must be within [0, 1] (`dynconfig.go:commonCodec`); `max_wait_time` cannot be less than `min_wait_time` (`dynconfig.go:WaitTime.Validate`).
- `Get` on a holder built for another type panics naming both types (`dynconfig.go:Get`); a nil holder reads as the zero value.
- `StartPoll`'s `onError` must not exit the process: the worker holds the last good value and one failed poll must not lose the run (`dynconfig.go:StartPoll`).

## Side Effects

- GETs the master's config endpoint over HTTP, on every spawn message and on a poll ticker (`dynconfig.go:SubscribeSpawn`, `dynconfig.go:StartPoll`).
- Pushes `trace_probability` changes into a `ProbabilityUpdater` sampler (`dynconfig.go:fetchAndApply`).
- Starts a polling goroutine (`dynconfig.go:StartPoll`); logs each applied change once (`dynconfig.go:fetchAndApply`).

## Verification

- Run: `go test ./internal/benchmarking/boomer/dynconfig/...`
