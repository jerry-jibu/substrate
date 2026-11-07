# boomer-worker

## Overview

Go re-implementation of the GluttonUser locust test. Speaks the locust
worker protocol via `myzhan/boomer`, so it appears as a regular worker to
the Python locust master while avoiding gevent's scheduling tax. User
classes register via `init()` from
`internal/benchmarking/boomer/{agentsession,ateenv,glutton,resumepaused,sweperf}`.

## Interface

- `main.go:main` - entry point. Flags include `--api-endpoint` (ateapi gRPC dial target, default `k8s:///api.ate-system.svc.cluster.local:443`), `--router-url` (must be https), `--atespace` (default `benchmark`), `--prometheus-addr` (default `:8001`), `--config-json`, `--master-web-port`, `--config-poll-interval`, `--user-class` (default `glutton`), `--actors-per-user`, `--check-agentsession-script`, `--total-actors`, `--spawn-concurrency`, `--actor-deadline`, `--http-max-idle-conns-per-host`.
- `main.go:main` selects the user class via `userclass.Lookup`, builds a `dynconfig.NewHolder` from the class codec, dials ateapi with `boomerutil.DialControl`, and runs `boomer.Run` until SIGINT/SIGTERM or master quit; then calls the class shutdown (suspend+delete actors) with a 60s budget.
- `--check-agentsession-script` validates an agent-session YAML (`agentsession.LoadFile`, `agentsession.Budgets`) and exits without running.
- Subdirectories: none.

## Invariants

- `--router-url` must start with `https://`; exit 1 otherwise (`main.go:main`).
- `--actors-per-user`, `--total-actors`, `--spawn-concurrency` must be >= 1 and `--actor-deadline` > 0; exit 1 otherwise (`main.go:main`).
- The first dynconfig fetch from the master must succeed or the worker exits so the pod restarts; later fetch failures keep the last good values (`main.go:main` `dynconfig.SubscribeSpawn` callback).
- Logging is JSON to stdout (`main.go:main` `slog.NewJSONHandler`).

## Side Effects

- Dials the ateapi Control gRPC endpoint (`main.go:main` `boomerutil.DialControl`) and speaks HTTPS to the atenet router with a transport keeping up to `--http-max-idle-conns-per-host` idle conns per host, HTTP/1.1 only, 5 minute idle timeout (`main.go:main`).
- Serves Prometheus metrics on `--prometheus-addr` (`main.go:main` `bmetrics.Serve`).
- Initializes OTel tracing via `btrace.Init` (`main.go:main`).
- Creates/hibernates/deletes actors in the atespace through the selected user class (`main.go:main` `entry.Init` shutdownFn).
- Optionally fetches dynconfig from `http://{master-host}:{master-web-port}/boomer-config` on spawn and per `--config-poll-interval` (`main.go:main`).

## Verification

- Run: `go test ./cmd/benchmarking/boomer-worker/...`
