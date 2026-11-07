# sandboxruntimes

## Overview

Reads and writes a worker's sandbox runtimes file, which atelet keeps in
the worker's directory as `nodepath.WorkerSandboxRuntimesPath`
(`nodepath.SandboxRuntimesFileName`); the worker registers its content as
`RegisterWorkerRequest.sandbox_classes`.

## Interface

- `sandboxruntimes.go:File` - the file content: one `sandboxruntimes.go:Class` per sandbox class the pool declares.
- `sandboxruntimes.go:Class` - `SandboxClass` (e.g. `gvisor`), `DefaultRuntime` (what new and cold-booted actors start under), `RestorableRuntimes` (other enabled versions in spec order, none repeating the default).
- `sandboxruntimes.go:Runtime`/`Attributes` - a runtime's compatibility attribute map (e.g. `sandbox_bundle_hash`).
- `sandboxruntimes.go:Write` - atomically replaces the file (temp file `sandboxruntimes.go:tempPrefix` + rename) unless content is unchanged; reports whether it wrote.
- `sandboxruntimes.go:Read` - decodes with `DisallowUnknownFields`; a missing file wraps `fs.ErrNotExist`.

## Invariants

- `RestorableRuntimes` never repeats another runtime or the default (`sandboxruntimes.go:Class` doc).
- Write is atomic (temp file + rename) and skips when the encoded content equals the current file, which is well-defined because `encoding/json` sorts map keys (`sandboxruntimes.go:Write`).
- Read rejects unknown fields, so the two sides of the file cannot drift silently (`sandboxruntimes.go:Read`).

## Side Effects

- Creates a temp file in the target directory and renames it over the file (`sandboxruntimes.go:Write`); reads the file (`sandboxruntimes.go:Read`).

## Verification

- Run: `go test ./internal/sandboxruntimes/...`
