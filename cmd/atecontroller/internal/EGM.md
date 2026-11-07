# internal

## Overview
Packages private to ate-controller: the controller-runtime reconcilers and the worker Pod syncer.

## Subdirectories & Boundaries
- `controllers`: `./controllers/EGM.md` - reconcilers for WorkerPool, NetworkPolicy, SandboxConfig protection, and egress MITM trust.
- `workersync`: `./workersync/EGM.md` - syncs worker Pods into the ateapi Worker registry.
