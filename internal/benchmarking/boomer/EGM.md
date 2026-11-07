# boomer

## Overview

Go implementations of the locust worker tasks (boomer, ZMQ protocol) for the benchmarking suites. Each subdirectory registers one or more user classes in the shared registry; the boomer worker binary selects a class by the `--user-class` flag and runs its task function with the shared dependencies in `userclass.Config`.

## Subdirectories & Boundaries

- [agentsession](./agentsession/EGM.md) - AgentSessionUser: scripted coding-agent sessions measuring wake latency, plus turnover mode.
- [ateenv](./ateenv/EGM.md) - AteEnvUser: ate-env actors executing processes over the ateenv gRPC API.
- [boomerutil](./boomerutil/EGM.md) - shared mTLS dialing, failure classification, retries, and telemetry helpers.
- [dynconfig](./dynconfig/EGM.md) - runtime-mutable locust-flag config fetched from the master over HTTP.
- [glutton](./glutton/EGM.md) - GluttonUser, SpawnUser, and DurdirUser against glutton actors.
- [metrics](./metrics/EGM.md) - Prometheus surface mirroring the Python locust workers, plus the boomer stats bridge.
- [resumepaused](./resumepaused/EGM.md) - ResumePausedUser: paced cold resume/suspend cycles against ateapi.
- [sweperf](./sweperf/EGM.md) - SweperfUser: SWE-bench trajectory replay partitioned into cycles.
- [trace](./trace/EGM.md) - OTLP tracing setup with a runtime-updatable sampler.
- [userclass](./userclass/EGM.md) - the user-class registry and the shared Config handed to every class.
