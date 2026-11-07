# isolate

## Overview

The isolate benchmark's fake data plane: two programs that stand in for
atecontroller's worker syncer and atelet in control-plane benchmarks, plus
the package they share. No actor is ever run; a fake Worker is a
registration in ate-api-server with no pod behind it.

## Subdirectories & Boundaries

- fake-atelet: [./fake-atelet/EGM.md](./fake-atelet/EGM.md) - DaemonSet stand-in for atelet; delayed Atelet success responses and the capacity relay.
- fake-workersync: [./fake-workersync/EGM.md](./fake-workersync/EGM.md) - stand-in for atecontroller's WorkerPool controller and worker syncer; reconciles fake Workers from WorkerPool replicas.
- internal: [./internal/EGM.md](./internal/EGM.md) - shared fake-worker pod naming, placement, and relay identity checks.
