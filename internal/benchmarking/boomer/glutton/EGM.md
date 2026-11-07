# glutton (boomer user classes)

## Overview

Three boomer-Go user classes driving glutton actors. GluttonUser cycles actors through resume, working-set fill/churn/read, CPU load, ping, and suspend. SpawnUser creates one batch of actors per process and records readiness times. DurdirUser asserts disk durability across suspend/resume by writing and digest-verifying a file. All three talk to ateapi over gRPC and to the actors over protobuf-through-HTTP via the atenet router.

## Interface

- `lifecycle.go:init` registers `Entry{Name: "glutton", UserClass: "GluttonUser", Config: gluttonCodec, Init: initPing}`.
- `lifecycle.go:taskRuntime.iterate()` / `shutdown(ctx)` - the GluttonUser task: per cycle `resume`, `ensureRAMFilled`, `ensureCPULoad`, `readRAM`, `churnRAM`, ping loop, `hibernate`.
- `lifecycle.go:gluttonActor` - per-actor state (`crashed`, `hibernatePending`, `deleted`) and lifecycle methods (`create`, `resume` with `ResumeActorFirstResume` first-resume row, `pause`, `suspend`, `delete`, `noteFailure`/`noteSuccess`).
- `spawn.go:init` registers `Entry{Name: "spawn", UserClass: "SpawnUser", Config: spawnCodec, Init: initSpawn}`; `spawn.go:spawnRuntime` runs one batch (`runBatch`, `runOneActor`, `createActorWithRetry`, `resumeActorWithRetry`, `pingUntilReady`, `deleteAll`); `spawn.go:firedDeciles(rank, totalActors)`.
- `durdir.go:init` registers `Entry{Name: "durdir", UserClass: "DurdirUser", Config: durDirCodec, Init: initDurDir}`; `durdir.go:durDirUser.step/bootstrap/writeDisk/readDisk` with routes `/writedisk`, `/readdisk`.
- `knobs.go:gluttonKnobs`, `knobs.go:durDirKnobs`, `knobs.go:spawnKnobs` - the three runtime-config slices with their codecs (`gluttonCodec`, `durDirCodec`, `spawnCodec`).
- `lifecycle.go` route constants: `/ping`, `/writeram`, `/readram`, `/usecpu`.

## Invariants

- Three consecutive `ReplaceIfPersistent` failures replace an actor; `RetryLater` errors never count, and a crashed actor is replaced on the spot (`lifecycle.go:noteFailure`, `lifecycle.go:maxConsecutiveFailures`).
- A failed suspend leaves `hibernatePending`; the next iteration finishes the re-entrant suspend before resuming, rather than waking a stranded actor (`lifecycle.go:iterate`).
- RAM fill, CPU load, and disk bootstrap run once per actor and are retried on failure; glutton keeps the allocations across suspend/resume (`lifecycle.go:ensureRAMFilled`, `lifecycle.go:ensureCPULoad`, `durdir.go:bootstrap`).
- Every Durdir read verifies the size and sha256 against the last write; DATA mode also recomputes the digest over the returned payload (`durdir.go:readDisk`).
- Spawn retries with 50ms-to-1s backoff plus jitter until the per-actor deadline; AlreadyExists on a create retry is the earlier attempt having landed, and only listed codes are terminal (`spawn.go:createActorWithRetry`, `spawn.go:isSpawnTerminalError`).
- `TimeToAllReady` carries the max, not last, ready elapsed; percentile rows fire when an actor's readiness rank matches the decile threshold (`spawn.go:runBatch`, `spawn.go:firedDeciles`).
- Spawn's `iterate` never returns after the batch: boomer calls it in a loop and returning would spin on the spent `runOnce` (`spawn.go:iterate`).
- `durdir_file_size_bytes` cannot exceed 2 GiB (`knobs.go:durDirCodec`); `cpu_duty_cycle` must be within [0, 1] (`knobs.go:gluttonCodec`).
- Durdir teardown hibernates before deleting; the hibernate is unmetered, the delete is metered (`durdir.go:hibernateAndDelete`).

## Side Effects

- Calls ateapi RPCs: `CreateAtespace` (AlreadyExists is success), `CreateActor`, `ResumeActor` (with `RetryOnConflict`), `SuspendActor` (sync/async per `lifecycle_mode`), `DeleteActor`.
- POSTs protobuf glutton RPCs (`Ping`, `WriteRAM`, `ReadRAM`, `UseCPU`, `WriteDisk`, `ReadDisk`) through the atenet router with the `atenet.TargetActorHeader` header.
- Records locust stats rows: `GluttonPing`, `GluttonFillRAM`, `GluttonUseCPU`, `GluttonChurnRAM`, `GluttonReadRAM`, `ActorTimeToReady`, `TimeToReady_<k>pct`, `TimeToAllReady`, `DurDirWrite`, `DurDirServeInitial`, `DurDirServeAfterResume`, `DurDirServeWarm`, `DurDirOverwrite`, `CrashCount`, plus the gRPC control rows.
- Spawns batch goroutines (spawn concurrency) and a 32-way delete sweep (`spawn.go:runBatch`, `spawn.go:deleteAll`).

## Verification

- Run: `go test ./internal/benchmarking/boomer/glutton/...`
