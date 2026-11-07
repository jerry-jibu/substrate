# atunnel benchmark

Measures what atunnel, the per-worker ingress and egress tunnel, costs in CPU
and memory as load rises. atunnel runs in a process of its own, hosting actors
with the worker's own code, so the numbers are atunnel's alone. `run-docker.sh`
runs the whole thing as three containers (load generator, privileged worker,
sink) on one Docker network. See `README.md` for the full scenario and caveats
write-up.

## Interface

- `run-docker.sh` - builds the linux binaries and one image, writes the PKI, runs the three containers, and saves results under `runs/<timestamp>/`; options `--tunnel-cpus`, `--tunnel-memory`, `--actors`, `--cpuset-worker`, `--cpuset-rig`, everything after `--` goes to `atunnel-bench load`.
- `Dockerfile` - distroless static image holding the single `atunnel-bench` binary; each container picks its subcommand.
- [cmd](./cmd/EGM.md) - the `atunnel-bench` command tree.
- [internal](./internal/EGM.md) - the benchmark's shared packages: fake atelet, load generator, PKI, process stats, report types, fake peers, and the worker-side host.

## Invariants

- The worker container is privileged with a private cgroup namespace, because atunnel creates real network namespaces, nftables rules, and cgroups; atunnel itself runs in a cgroup with the CPU and memory limits and no swap (`run-docker.sh` worker invocation, `cmd/atunnel-bench/worker.go`).
- The worker's credentials are mounted where a worker Pod's are projected (`/run/podidentity.podcert.ate.dev/...`), so atunnel runs on its default flags (`run-docker.sh` worker mounts).
- The generator and the egress actor containers set `net.ipv4.ip_local_port_range=1024 65535` and `tcp_tw_reuse=1`, because Connect and EgressNewConnection leave tens of thousands of sockets in TIME_WAIT (`run-docker.sh` `port_opts`, `cmd/atunnel-bench/sandbox.go:runEgressActor`).
- The actors' sockets travel through a named Docker volume, not a host mount, because sockets do not work on Docker Desktop's host mounts (`run-docker.sh` `shared`).

## Side Effects

- `run-docker.sh` builds Go binaries and a Docker image, creates and removes a Docker network and volume, runs privileged containers, and writes `runs/<timestamp>/{results.txt,results.json,profiles/,logs/}` (`run-docker.sh`).
- Only `results.txt` and a hand-written `report.md` are kept in git; `runs/` otherwise holds run artifacts (`run-docker.sh` output layout, `.gitignore`).

## Verification

- Run: `go test ./benchmarking/atunnel/...` from the repo root.
