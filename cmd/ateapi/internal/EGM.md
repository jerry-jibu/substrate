# internal

## Overview
Packages private to the ateapi binary: the Control and Worker service implementations, authentication, authorization, persistence and its PostgreSQL backend, worker caching, scheduling, validation, and field defaults.

## Subdirectories & Boundaries
- `controlapi`: `./controlapi/EGM.md` - implements the Control gRPC service over the store and atelets.
- `workerservice`: `./workerservice/EGM.md` - serves the WorkerService RPCs workers call.
- `store`: `./store/EGM.md` - persistence contract and the PostgreSQL implementation under `atepg`.
- `authz`: `./authz/EGM.md` - OpenFGA-backed authorization for every served RPC.
- `apiauthn`: `./apiauthn/EGM.md` - authenticates Control RPC clients (SPIFFE mTLS, OIDC JWTs).
- `ateletauth`: `./ateletauth/EGM.md` - authenticates RPCs arriving from atelets.
- `scheduling`: `./scheduling/EGM.md` - picks the worker that hosts an actor.
- `workercache`: `./workercache/EGM.md` - in-memory view of all workers.
- `apivalidation`: `./apivalidation/EGM.md` - request and field validation for Control and WorkerService RPCs.
- `defaults`: `./defaults/EGM.md` - applies resource field defaults in place.
