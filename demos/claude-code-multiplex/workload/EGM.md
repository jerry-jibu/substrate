# claude-code-multiplex/workload

## Overview

Demo actor workload: a container image that periodically invokes Claude Code
with a prompt and idles between ticks. The idle window is what Substrate uses
to suspend the actor and multiplex its worker onto another actor.

## Interface

- `run.sh` entrypoint (`run.sh:38`): loops forever, running
  `claude --print "${TASK}"` each tick and sleeping `INTERVAL_SECONDS`
  between ticks.
- Environment contract (`run.sh:21`-`:24`): `TASK` (prompt), 
  `INTERVAL_SECONDS` (idle length), `ANTHROPIC_API_KEY` (required),
  `ACTOR_NAME` (defaults to the hostname, `run.sh:33`).
- `Dockerfile`: `node:20-slim` base, installs
  `@anthropic-ai/claude-code@latest`, sets `TASK` and `INTERVAL_SECONDS`
  defaults, entrypoint `/run.sh` (`Dockerfile:15`-`:29`).

## Invariants

- The workload refuses to start without `ANTHROPIC_API_KEY`
  (`run.sh:28`-`:31`).
- A non-zero `claude` exit does not kill the loop; the tick is logged and
  the workload keeps idling (`run.sh:46`).

## Side Effects

- Makes outbound API calls to Anthropic via the `claude` CLI on every tick
  (`run.sh:46`); those traverse the actor's egress path.
- Writes tick logs to stdout for `kubectl logs` (`run.sh:40`-`:48`).

## Verification

- None configured for this container workload; the image is built and run by
  the demo's install templates.
