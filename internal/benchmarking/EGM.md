# benchmarking

## Overview

Load-testing support library for Agent Substrate: the glutton in-sandbox workload actor and the boomer-Go locust user classes that drive actors through the ateapi control plane and the atenet router.

## Subdirectories & Boundaries

- [boomer](./boomer/EGM.md) - Go locust worker user classes (agentsession, ateenv, glutton, resumepaused, sweperf) plus the shared registry, dynconfig, metrics, tracing, and helper packages.
- [glutton](./glutton/EGM.md) - the glutton workload service (RAM/disk/CPU/FD consumption, gossip, scripts) served by `cmd/benchmarking/glutton`, and its httptest fake.
