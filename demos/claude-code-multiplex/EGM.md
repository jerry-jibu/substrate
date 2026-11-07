# claude-code-multiplex

## Overview

Demo showing worker multiplexing with Claude Code agents. A two-pod
`WorkerPool` is shared by three agent `ActorTemplate`s, so the substrate's
suspend/resume behavior becomes visible as tasks queue for a free worker.
The install templates create the namespace and pool; the agent templates are
created through the ate API with `kubectl ate create actor-template`
(`claude-code-multiplex.yaml.tmpl:15`-`:19`).

## Interface

- `claude-code-multiplex.yaml.tmpl`: Namespace `claude-multiplex-demo` plus
  `WorkerPool/claude-workerpool` with 2 replicas of
  `ko://go.ate.dev/substrate/cmd/ateworker-gvisor`, 1 CPU / 2560Mi limits
  per worker, sandbox class `gvisor` (`claude-code-multiplex.yaml.tmpl:19`-`:53`).
- `agent-orion-template.yaml.tmpl`, `agent-mars-template.yaml.tmpl`,
  `agent-luna-template.yaml.tmpl`: the three agent ActorTemplates sharing the
  pool (see `claude-code-multiplex.yaml.tmpl:16`-`:17`).
- ui: [./ui/EGM.md](./ui/EGM.md) - visualization server driving task
  assignment and actor suspend/resume.
- workload: [./workload/EGM.md](./workload/EGM.md) - the Claude Code tick
  container the agents run.

## Invariants

- The pool's per-worker CPU limit matches one agent template's 1 CPU, so the
  2-pod pool hosts at most 2 running agents at once; the memory limit sits
  above the agent's because the worker keeps some for itself
  (`claude-code-multiplex.yaml.tmpl:39`-`:47`).
- Three agents share the two workers, which is the multiplex pressure the
  demo exists to show (`claude-code-multiplex.yaml.tmpl:15`-`:19`).

## Side Effects

- Applying the templates creates a namespace, a WorkerPool, and its worker
  Deployment in the target cluster; the UI's reconcile loop then suspends
  and resumes real actors (`ui/server.go:285`).

## Verification

- Run: `go test ./demos/...`
