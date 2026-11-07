# ate-setup/internal/demos/claudemultiplex

## Overview

Installs the claude-code-multiplex demo: several Claude Code agents multiplexed
onto one WorkerPool. Its workload is a Dockerfile-based node:20-slim and Claude
Code wrapper rather than a Go binary, so the image is built with docker buildx
and the resolved digest is substituted into the agent templates
(`claudemultiplex.go` package doc).

## Interface

- `demo` (`claudemultiplex.go`) - implements `demos.Demo` by wrapping (not embedding) a `demos.Substrate` named `sub`, so the shared render test skips its deploy-time placeholders.
- `agentTemplates()`, `agents`, `anthropicEgress` (`claudemultiplex.go`) - the three agent ActorTemplates (luna, mars, orion) and a TLS-passthrough EgressRule for `api.anthropic.com:443`.
- `Deploy` (`claudemultiplex.go`) - requires `ANTHROPIC_API_KEY` and `KO_DOCKER_REPO`, deploys the substrate, then creates each actor and its egress policy via `steps.CreateActor` / `steps.CreateActorEgressPolicy`.
- `renderValues` (`claudemultiplex.go`) - builds the workload image with `images.BuildDockerfileImage` and returns the `ANTHROPIC_API_KEY` and `WORKLOAD_IMAGE` placeholder values.

## Invariants

- Deploy fails fast when `e.Cfg.AnthropicAPIKey` or `e.Cfg.KODockerRepo` is empty (`claudemultiplex.go` `Deploy`).
- Delete needs no credentials: only the pool manifest is rendered, and it carries none of the deploy-time placeholders (`claudemultiplex.go` `Delete`).
- Reports no settings of its own; `ANTHROPIC_API_KEY` is the installer's `demo.anthropicAPIKey` setting (`claudemultiplex.go` `Settings`).

## Side Effects

- Builds and pushes a Dockerfile image through `docker buildx` (`images.BuildDockerfileImage`).
- Creates actors and egress policies through the ate API client (`steps.CreateActor`, `steps.CreateActorEgressPolicy`).
- Deploys a namespace, WorkerPool, and ActorTemplates via the shared `demos.Substrate.Deploy`.

## Verification

- Run: `go test ./cmd/ate-setup/internal/demos/claudemultiplex/...`
