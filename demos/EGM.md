# demos Blueprint

## System Overview

Self-contained example applications for Agent Substrate. Each demo is one
directory holding an optional workload binary, install templates
(`*.yaml.tmpl` for the WorkerPool and namespace, `*-template.yaml.tmpl` for
the ActorTemplate, which is created through the ate API with
`kubectl ate create actor-template`, not applied as a CRD), and any driver
scripts. All Go code belongs to the root module, `go.ate.dev/substrate`.

## Subdirectories & System Boundaries

- claude-code-multiplex: [./claude-code-multiplex/EGM.md](./claude-code-multiplex/EGM.md) - three
  Claude Code agents sharing two workers, with a UI visualizing the
  suspend/resume multiplexing.
- counter: [./counter/EGM.md](./counter/EGM.md) - the primary demo; memory,
  durable-volume, and rootfs counters observable across snapshots.
- egress: [./egress/EGM.md](./egress/EGM.md) - per-actor egress policy
  demo with HTTP, gRPC, WebSocket, and CONNECT endpoints plus an identity
  test script.
- multi-template: [./multi-template/EGM.md](./multi-template/EGM.md) - two
  ActorTemplates with different binaries sharing one WorkerPool.
- parking: [./parking/EGM.md](./parking/EGM.md) - request-parking under
  oversubscription, driven by `load.sh`.
- sandbox: [./sandbox/EGM.md](./sandbox/EGM.md) - stateless command-running
  actor and its REPL client.
- jupyter: not documented here (template-only demo, no code).

## Global Invariants

- Workloads listen on port 80 and expose a wakeup-probe-compatible readiness
  path (for example counter's `/readyz`, `counter/counter.go:128`), matching
  what `WakeupProbe` in `api/ateapi/v1/ateapi.proto:1453` polls.
- Worker images in install templates are referenced ko-style, for example
  `ko://go.ate.dev/substrate/cmd/ateworker-gvisor`
  (`counter/counter.yaml.tmpl:40`).
- ActorTemplates are Substrate API resources created with `kubectl ate`,
  never Kubernetes CRDs (`counter/counter.yaml.tmpl:15`-`:17`).

## Global Side Effects

- Demos create namespaces, WorkerPools, actors, and local port-forward
  proxies in the target cluster when installed or driven
  (`parking/load.sh:101`-`:111`, `egress/test-egress.sh:26`-`:32`).

## System Verification

- Run: `go test ./demos/...`
