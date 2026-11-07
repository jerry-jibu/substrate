# benchmarking

## Overview

Benchmark binaries: workload generators and fake data planes for
control-plane and egress scale tests.

## Subdirectories & Boundaries

- boomer-worker: [./boomer-worker/EGM.md](./boomer-worker/EGM.md) - Go locust worker driving actor benchmarks via myzhan/boomer.
- egress: [./egress/EGM.md](./egress/EGM.md) - `benchmark-egress`, the actor and target of the egress scale test.
- glutton: [./glutton/EGM.md](./glutton/EGM.md) - gRPC/HTTP workload that consumes RAM, disk, and file descriptors.
- isolate: [./isolate/EGM.md](./isolate/EGM.md) - fake data plane (fake-atelet, fake-workersync) for control-plane benchmarks.
