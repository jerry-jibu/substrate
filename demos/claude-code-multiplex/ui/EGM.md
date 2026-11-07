# claude-code-multiplex/ui

## Overview

Demo UI server for the Claude Code multiplexing demo. A stdlib HTTP server
(`server.go`) that reads worker and actor state from the ateapi Control
service, streams pod logs through client-go, and serves `index.html`, which
polls its JSON endpoints. It simulates task assignments and drives
suspend/resume of the demo's agent actors to make worker multiplexing
visible.

## Interface

- `main` (`server.go:629`): connects to ateapi via
  `ateclient.NewClient` (`server.go:638`), starts a one-second
  `reconcileAgents` ticker (`server.go:656`-`:663`), and serves on
  `PORT` (default 8080).
- HTTP endpoints (`server.go:666`-`:672`): `/` and `/index.html`
  (`handleIndex`), `/healthz` (`handleHealthz`), `/api/pods`
  (`handlePods`, from `ateapi.ControlClient.ListWorkers`), `/api/actors`
  (`handleActors`, from `ListActors`), `/api/logs/` (`handleLogs`, via
  client-go `Pods.GetLogs` with a 25-line tail), `/api/task-status`
  (`handleTaskStatus`), and POST `/api/give-task` (`handleGiveTask`).
- `reconcileAgents` (`server.go:285`): suspends idle running agents, resumes
  waiting agents only into a free worker slot, and never suspends an agent
  mid-task.
- Environment: `PORT`, `DEMO_NAMESPACE` (default
  `claude-multiplex-demo`), `ATEAPI_ADDR` (empty means auto port-forward to
  `svc/api` in `ate-system`) (`server.go:69`-`:71`, `:136`-`:137`).

## Invariants

- Assignments move `queued` -> `running` -> `completed`, and a task leaves
  `queued` only once `reconcileAgents` starts it (`server.go:76`-`:82`,
  `computeState` at `:257`).
- At most `maxAssignments` (50) assignments are kept, newest first
  (`server.go:72`, `:452`-`:455`).
- `reconcileAgents` never suspends an agent with a running or waiting task,
  and resumes only while `occupied < poolSize` (`server.go:285`-`:363`).
- A missing kube context leaves `kubeClient` nil and `/api/logs` degrades to
  503 instead of crashing (`server.go:183`-`:198`, `:578`).

## Side Effects

- Calls the ateapi Control service: `ListActors`, `ListWorkers`,
  `ResumeActor`, `SuspendActor` (`server.go:239`, `:381`, `:368`, `:400`).
- Reads Kubernetes pod logs via client-go (`server.go:586`).
- `ateclient.NewClient` with an empty address port-forwards to `svc/api`,
  mints an ate-client ServiceAccount token, and verifies the serving cert
  against the live ClusterTrustBundle (`server.go:629`-`:644`).
- Mutates the in-memory `assignments` slice under `mu`
  (`server.go:135`-`:141`).

## Verification

- Run: `go test ./demos/...`
