# ateapi

## Overview

API group for the Substrate Control plane gRPC API. The `v1` directory holds
the `substrate.ateapi.v1` proto package, its generated Go bindings, and the
embedded proto source used by tooling.

## Subdirectories & Boundaries

- v1: [./v1/EGM.md](./v1/EGM.md) - the `Control` and `WorkerService` gRPC
  services, resource messages, and generated Go stubs.
