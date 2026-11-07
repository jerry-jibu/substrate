# workerservice

## Overview
Serves the ateapi WorkerService RPCs a worker calls to register itself and report what it observes ("Package workerservice", `service.go`). Worker requests run through the same control entry points Control uses, so they are subject to the same preconditions.

## Interface
- `service.go:Server` implements `ateapi.WorkerServiceServer` (compile-checked by `var _ ateapi.WorkerServiceServer = (*Server)(nil)`), built with `service.go:New`.
- RPCs: `register.go:Server.AcquireWorkerEpoch`, `register.go:Server.RegisterWorker`, `register.go:Server.SetWorkerSandboxRuntimes`, `suspend.go:Server.RequestActorSuspend`, `report.go:Server.ReportActorSnapshotState`, `crash.go:Server.ReportActorCrashed`, `jwt.go:Server.MintWorkerActorJwt`, `certificate.go:Server.MintWorkerActorCertificate`.
- `service.go:actorControl` is the in-process slice of Control this package drives; `*controlapi.RPCService` satisfies it.

## Invariants
- Every worker request is authorized against the authoritative store state (`service.go:Server.store` doc).
- Actor suspend, snapshot state, and crash reports go through `actorControl`, so a worker's request passes the same preconditions as a client's (`service.go:actorControl` doc).
- `service.go:Server` verifies the calling atelet's identity against `ateletSPIFFEID`.
- The actor JWT issuer and signing pool are the same ones `Control.MintActorJwt` uses (`service.go:actorJWTIssuer` doc).

## Side Effects
- Writes worker registrations and reports through `store.Interface` (`service.go:Server.store`).
- Mints worker-for-actor JWTs (`localjwtauthority.Pool`) and actor certificates (`localca.Pool`) (`service.go:New`).

## Verification
- Run: `go test ./cmd/ateapi/internal/workerservice/...`
