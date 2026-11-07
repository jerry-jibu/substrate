# userclass

## Overview

Registry of the boomer-Go load-test user classes and the shared dependency Config each class receives. A class registers an Entry at init; the worker binary looks the entry up by the `--user-class` flag value and calls its Init to get the boomer task function and shutdown hook.

## Interface

- `registry.go:Entry` - `Name` (the `--user-class` flag value), `LocustFile`, `UserClass` (must equal the boomer task name), `Config dynconfig.Codec`, `Init func(*Config) (task func(), shutdown func(context.Context))`.
- `registry.go:Add(e Entry)` - registers a class; panics on duplicate `Name`.
- `registry.go:Lookup(name) (Entry, bool)`; `registry.go:Names() []string` (sorted).
- `config.go:Config` - dependencies handed to Init: `APIStub ateapi.ControlClient`, `HTTPClient`, `RouterURL`, `Atespace`, `Dyn *dynconfig.Holder`, `Tracer`, plus spawn-benchmark knobs `ActorsPerUser`, `TotalActors`, `SpawnConcurrency`, `ActorDeadline`.

## Invariants

- `Entry.UserClass` must equal `boomer.Task.Name` or the master's spawn messages never match and no users start (`registry.go:Entry`).
- `Add` panics on a duplicate registration (`registry.go:Add`).
- A payload the class's codec refuses is rejected at fetch time, not mid-run (`registry.go:Entry.Config` doc).
- `Config.Dyn` is required: classes read their typed knobs via `dynconfig.Get` every iteration (`config.go:Config.Dyn`).
- `ActorsPerUser` of zero or negative is treated as 1 by classes that honor it (`config.go:Config.ActorsPerUser`).

## Side Effects

- Registry mutation is guarded by an RWMutex (`registry.go`).

## Verification

- Run: `go test ./internal/benchmarking/boomer/userclass/...`
