# atunnel-bench

Command `atunnel-bench` drives the atunnel benchmark. Every benchmark process
is a subcommand of this one program; `benchmarking/atunnel/run-docker.sh` shows
how the containers run them together.

## Interface

- `main.go:main` - cobra root command `atunnel-bench`; registers `pkiCommand`, `sinkCommand`, `loadCommand` (portable) and `linuxCommands` (Linux only), runs with a signal context.
- `pki.go:pkiCommand` - subcommand `pki`: writes the shared throwaway certificates via `pki.Generate`; flags `--dir` (required), `--worker-hosts`, `--gateway-hosts`, `--validity`.
- `sink.go:sinkCommand` / `sink.go:runSink` - subcommand `sink`: runs the fake egress gateway (`sink.NewGateway`, `sink.AdminHandler`); flags `--pki-dir` (required), `--gateway-listen` (`:9443`), `--admin` (`:9091`), `--body-bytes`.
- `load.go:loadCommand` - subcommand `load`: runs scenarios at rising rates via `load.Runner` and prints and saves reports; flags include `--pki-dir` (required), `--tunnel-host`, `--ingress-port`, `--connect-port`, `--tunnel-admin`, `--sink-admin`, `--shared-dir`, `--scenarios`, `--rungs`, `--step`, `--warmup`, `--workers`, `--profile-dir`, `--results-json`.
- `tunnel.go:tunnelCommand` / `tunnel.go:runTunnel` (Linux) - subcommand `tunnel`: the measured process; starts atunnel hosting actors via `workerhost.Start` with flags from `atunnel.RegisterFlags`, plus `--actors`, `--gateway`, `--admin` (`:9090`), `--log-level`; serves `/stats`, `/healthz`, and `/debug/pprof`.
- `worker.go:workerCommand` / `worker.go:supervisor.run` / `worker.go:supervisor.runOnce` (Linux) - subcommand `worker`: the rest of the worker pod; serves the fake atelet, starts `tunnel` in a cgroup with CPU and memory limits, places ingress and egress actors in the sandboxes, and replaces atunnel when it dies; flags `--pki-dir`, `--tunnel-cpus`, `--tunnel-memory`, `--shared-dir`, `--tunnel-admin`, `--body-bytes`; arguments after `--` go to `tunnel`.
- `sandbox.go:netnsExecCommand` / `sandbox.go:netnsExec` (Linux) - hidden subcommand `netns-exec NETNS COMMAND`: enters a network namespace (`unix.Setns` with `CLONE_NEWNET`) and execs COMMAND.
- `sandbox.go:ingressActorCommand` / `sandbox.go:runIngressActor` (Linux) - subcommand `ingress-actor`: HTTP server standing in for an actor receiving ingress, counters on a Unix socket; flags `--port` (default `workerhost.ActorPort`), `--body-bytes`, `--stats-socket` (required).
- `sandbox.go:egressActorCommand` / `sandbox.go:runEgressActor` (Linux) - subcommand `egress-actor`: runs egress loads inside a sandbox for `load`; flag `--socket` (required); serves `load.ServeEgressActor`.
- `commands_linux.go:linuxCommands` - registers `tunnel`, `worker`, `ingress-actor`, `egress-actor`, `netns-exec` on Linux.
- `commands_other.go:linuxCommands` - returns no commands on non-Linux builds, so those subcommands do not exist there.

## Invariants

- `load` rejects empty or nonpositive `--rungs` and requires `--warmup` shorter than `--step` (`load.go:loadCommand` RunE).
- `pki`, `sink`, `ingress-actor`, and `egress-actor` mark their certificate or socket directory flags required (`pki.go`, `sink.go`, `sandbox.go` `MarkFlagRequired`).
- The hosting subcommands build only on Linux: `tunnel.go`, `sandbox.go`, `worker.go`, and `commands_linux.go` carry `//go:build linux`, and `main.go:main` adds `linuxCommands()` to the root, which is empty elsewhere.
- `worker` starts each `tunnel` incarnation in its own cgroup with `memory.swap.max` set to 0, so exceeding `--tunnel-memory` is an OOM kill, as in a pod (`worker.go:supervisor.createCgroup`).
- `worker` waits for atunnel's `/stats` to report hosted sandboxes before starting the sandbox actors, and kills them when the rung ends (`worker.go:supervisor.waitForSandboxes`, `worker.go:supervisor.startSandboxProcesses`).
- `runEgressActor` widens `/proc/sys/net/ipv4/ip_local_port_range` and sets `tcp_tw_reuse` to 1 before serving, so long runs of short connections do not exhaust ports (`sandbox.go:runEgressActor`).
- `listenUnix` removes a stale socket file and chmods the new one to 0777 so the load generator, another user in another container, can connect (`sandbox.go:listenUnix`).

## Side Effects

- `netnsExec` replaces the process with the command via `syscall.Exec` after entering the namespace (`sandbox.go:netnsExec`).
- `worker` writes cgroup v2 files under `/sys/fs/cgroup` (`cgroup.procs`, `cgroup.subtree_control`, `cpu.max`, `memory.max`, `memory.swap.max`, `memory.events`), execs child processes, and appends each atunnel death, with OOM kills counted, to `tunnel-deaths.log` in `--shared-dir` (`worker.go:delegateCgroups`, `worker.go:supervisor.createCgroup`, `worker.go:supervisor.logDeath`).
- `tunnel` and `sink` listen on TCP admin addresses; `tunnel` also exposes Go pprof endpoints (`tunnel.go:runTunnel`, `sink.go:runSink`).
- `load` writes results JSON after every scenario via `report.Write` and fetches CPU and heap profiles into `--profile-dir` (`load.go:loadCommand`, delegated to `load.Runner`).
- `runIngressActor` and `runEgressActor` create and serve on Unix sockets under `--shared-dir` (`sandbox.go`).

## Verification

- Run: `go test ./benchmarking/atunnel/cmd/atunnel-bench/...` from the repo root.
