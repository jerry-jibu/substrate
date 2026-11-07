# load

The benchmark's load generator: an open-loop pacer, the traffic modes that
exercise each atunnel path, the scenarios built from them, and the ladder that
runs a scenario at rising rates while reading atunnel's own CPU and memory from
its admin endpoint. Egress traffic is generated inside an actor's sandbox by
the egress actor, driven over a Unix socket.

## Interface

- `modes.go:Mode` - traffic kinds: `IngressH1`, `IngressH2`, `Connect`, `EgressHTTP`, `EgressIdle`, `EgressStream` (`modes.go` constants).
- `modes.go:Targets` - where traffic goes and with what credentials: `Ingress`, `Connect`, `Egress`, `EgressDiscard` addresses, `ServerName`, `RouterBundle`, `CA`, cycled `Actors`, `ActorPort`.
- `modes.go:execute` - runs one `options` against `Targets` through the mode's driver and returns a `report.Load`; returns an error only when the run could not start.
- `scenarios.go:Scenario` / `scenarios.go:Scenarios` - the seven scenarios in run order: `IngressH1`, `IngressH2`, `Connect`, `EgressHTTP`, `EgressNewConnection`, `EgressIdleConns`, `EgressTCP`.
- `scenarios.go:Select` / `scenarios.go:Names` - select scenarios by comma-separated name or `all`; unknown names are an error.
- `pacer.go:pacer.run` - open-loop scheduling: units leave at a fixed rate whatever each takes, at most `workers` in flight, with a bounded queue (`maxPending` 10000); overfull units are skipped and counted.
- `ladder.go:Runner` - runs scenarios against a live atunnel and sink; fields set the admin URLs, step and warmup lengths, worker cap, profile directory, and shared socket directory.
- `ladder.go:Runner.Prepare` - waits for both processes, learns actors and the actor port from atunnel's `/stats`, waits for the sandbox actor sockets, and sends a short check down every traffic path so misconfiguration fails here.
- `ladder.go:Runner.Run` - runs one scenario across the rungs, settles until the sink's tunnels close, and returns the `report.Report`; waits for a replaced atunnel after a DOWN rung.
- `ladder.go:verdict` - classifies each rung: `OK`, `RIG-LIMITED`, `ERRORS`, `SATURATED`, `DOWN`, or `SHORTFALL`, from errors, CPU shares (`tunnelSaturated` 0.95, `rigSaturated` 0.85), skipped units, send lag, and achieved rate.
- `stats.go:Stats` - the JSON every benchmark process serves on `/stats`: `process` snapshot plus optional `tunnel`, `ingress_actor`, and `gateway` sections.
- `stats.go:startWatch` / `stats.go:watch.finish` - sample a process's `/stats` over the measured window for peaks and CPU cost; a process whose `StartedAt` changes counts as restarted, and after `failuresToGiveUp` (2) failed reads the watch calls `onGone`, which stops the load.
- `egress_actor.go:ServeEgressActor` - serves `GET /stats` and `POST /run` on a listener inside the sandbox; `POST /run` takes `egressActorRequest` (options on the wire) and streams the line `holding` plus a JSON `egressActorResult`.
- `egress_actor.go:executeOnEgressActor` / `egress_actor.go:executeOnEgressActors` - run options on the egress actor over its Unix socket, splitting an `EgressIdle` run evenly across one actor per sandbox.
- `egress_actor.go:mergeLoads` - sums side-by-side loads, taking the worst latency percentiles.

## Invariants

- The pacer is open loop: each unit carries its due instant, latency is measured from it, and a generator that falls behind shows it as latency and lag rather than less offered load (`pacer.go:pacer.run`, `pacer.go:unit`).
- Units are measured only outside warmup: `pacer.run` marks units by `due` against `warmup`, and `modes.go:run.done` drops unmeasured units (`pacer.go`, `modes.go`).
- Egress modes run inside a sandbox, dialed to the fixed destinations `egressAnswerDestination` (`192.0.2.10:80`) and `egressDiscardDestination` (`192.0.2.10:9`), which nothing answers; atunnel takes every connection (`egress_actor.go` constants, `egress_actor.go:ServeEgressActor`).
- `EgressIdleConns` rungs are connection counts, opened at `idleOpenRate` (200/s), with every open measured and no warmup (`scenarios.go:idleOpenRate`, `ladder.go:Runner.rung`).
- `EgressTCP` runs one rung only, 4 connections (`streams`), 64 KiB writes (`ladder.go:Runner.rung`, `scenarios.go:Scenarios`).
- A rung where atunnel stops answering ends the scenario with verdict `DOWN`; the runner then waits for the replacement and fails if it does not come back (`ladder.go:Runner.Run`, `ladder.go:Runner.rung`).
- When atunnel dies, the sink and generator readings for that rung are dropped, because the worker replaces those actors with atunnel (`ladder.go:Runner.rung`).
- `verdict` only calls a rung `SATURATED` from atunnel's own CPU against its cgroup or GOMAXPROCS limit (`ladder.go:verdict`, `ladder.go:cpuLimit`).
- `sinkDelta` leaves out counters that went down, as one reset by a replaced ingress actor was (`ladder.go:sinkDelta`).

## Side Effects

- Network: ingress HTTPS to `Targets.Ingress`, CONNECT tunnels to `Targets.Connect`, plain HTTP and raw TCP to the egress destinations (`modes.go` drivers).
- Reads admin endpoints over TCP and Unix sockets: `/stats`, `/stats?gc=1`, `/healthz`, and pprof profiles, saving profiles to files (`stats.go:admin`, `stats.go:admin.fetch`).
- `ServeEgressActor` and the actors' stat sockets serve on Unix sockets in `SharedDir` (`egress_actor.go:serveEgressActor`, `stats.go:unixAdmin`).
- `Runner.Run`'s results are printed by the caller and written to disk by `report.Write`; profiles land in `Runner.ProfileDir` as `<Scenario>-<rung>.{cpu,heap}.pb.gz` (`ladder.go:profileName`).

## Verification

- Run: `go test ./benchmarking/atunnel/internal/load/...` from the repo root.
