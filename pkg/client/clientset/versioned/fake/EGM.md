# clientset/versioned/fake

## Overview
Generated fake clientset for tests (`client-gen` output; `clientset_generated.go`, `register.go`, and `doc.go` carry `Code generated ... DO NOT EDIT`). Implements the versioned `clientset.Interface` against an in-memory object tracker.

## Interface
- `NewSimpleClientset(objects ...runtime.Object) *Clientset` seeds the tracker with the given objects (`clientset_generated.go:NewSimpleClientset`)
- `Clientset` embedding `testing.Fake`, with `Discovery()`, `Tracker()`, `AteV1()`, and `IsWatchListSemanticsUnSupported() == true` (`clientset_generated.go:Clientset`)
- Package-level `scheme`, `codecs`, and `AddToScheme` mirroring the real clientset's scheme registration (`register.go:scheme`, `register.go:AddToScheme`)

## Invariants
- The tracker applies creates, updates, and deletions as-is, without field management, validation, or defaulting; it is meant for simple unit tests, not as a real clientset replacement (`clientset_generated.go:NewSimpleClientset` doc comment).
- `IsWatchListSemanticsUnSupported` returns true, so the reflector never uses WatchList semantics against the fake (`clientset_generated.go:IsWatchListSemanticsUnSupported`).

## Side Effects
- None beyond the in-memory object tracker and watch reactors registered by `NewSimpleClientset` (`clientset_generated.go:NewSimpleClientset`).

## Verification
- Run: `go test ./pkg/client/clientset/versioned/fake/...` from the repo root.
