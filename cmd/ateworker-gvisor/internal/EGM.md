# internal

## Overview
Packages private to the gVisor ateworker binary: CDI bundle injection and cgroup v2 stats reading.

## Subdirectories & Boundaries
- `cdiinject`: `./cdiinject/EGM.md` - applies CDI device edits to an actor's OCI bundle.
- `cgroupstats`: `./cgroupstats/EGM.md` - reads cgroup v2 usage counters.
