# nodepath

## Overview

The node-level paths more than one component needs: the host directory
atelet and workers both mount, the sockets and netns names they find each
other by, the per-actor parent directories, and where atelet stages runtime
binaries. This is one of the two surfaces atelet and the worker runtimes may
share (with `internal/proto/ateworkerpb`).

## Interface

- `nodepath.go:BasePath` - `/var/lib/ate`, the root shared folder mounted at the same path into atelet and worker containers.
- `nodepath.go:ActorsDir`, `nodepath.go:SnapshotsDir` - per-actor and per-snapshot parent dirs; snapshots separate so it can mount its own disk.
- `nodepath.go:UnderBasePath` - whether a dir resolves strictly under `BasePath`.
- `nodepath.go:ActorSandboxAssetsDir` - `actors/<uid>/sandbox-assets`, the only location a worker runs a sandbox binary from; both sides derive it from the actor UID so a worker can verify a request's dir.
- `nodepath.go:WorkerSupportSocket` - node-local atelet socket atunnel uses to request credentials.
- `nodepath.go:AteletOTLPSocketPath` - node-scoped unix socket `atelet-otlp.sock` for the OTLP relay.
- `nodepath.go:WorkersDir`/`WorkerPath`/`WorkerSocketPath` - `workers/<podUID>/worker.sock`; a dir may exist before its worker listens.
- `nodepath.go:SandboxRuntimesFileName` (`sandbox-runtimes.json`) and `nodepath.go:WorkerSandboxRuntimesPath` - the sandbox runtimes file atelet writes per worker pod.
- `nodepath.go:ActorNetNSName`/`ActorNetNSPath` - `worker-actor:<uid>` named netns under `/run/netns`.

## Invariants

- Both atelet and workers derive `ActorSandboxAssetsDir` from the actor UID independently; a worker must reject a request whose `sandbox_assets_dir` does not match (`nodepath.go:ActorSandboxAssetsDir` doc).
- A listed sandbox runtime may exist without its assets being on the node yet (`nodepath.go:SandboxRuntimesFileName` doc).
- `BasePath` is mounted writable and shared, so a worker pod can unlink or replace `AteletOTLPSocketPath`; that is a property of the whole mount, documented not solved (`nodepath.go:AteletOTLPSocketPath` doc).

## Side Effects

- ["None"] (pure path computation; the files and sockets named here are created by atelet and the workers, not by this package)

## Verification

- Run: `go test ./internal/nodepath/...`
