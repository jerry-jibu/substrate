# controlapi

## Overview
Implements the ateapi Control gRPC service: the control plane API for actors, workers, tags, actor templates, egress policies, access policies, and atespaces. `service.go:RPCService` is the thin gRPC shell; `service.go:ServiceImpl` holds business logic; `workflow.go:ActorWorkflow` and `workflow.go:WorkerWorkflow` orchestrate actor and worker lifecycle against the store and atelets.

## Interface
- `service.go:RPCService` implements `ateapi.ControlServer` (compile-checked by `var _ ateapi.ControlServer = (*RPCService)(nil)`), built with `service.go:NewRPCService`.
- Actor RPCs: `actor.go:RPCService.CreateActor`, `GetActor`, `ListActors`, `UpdateActor`, `DeleteActor`, `ResumeActor`, `SuspendActor`, `RevertActor`, `MintActorJwt`, `CrashActorReportedByWorker`.
- Worker RPCs: `worker.go:RPCService.CreateWorker`, `UpdateWorker`, `DeleteWorker`, `DrainWorker`, `MarkWorkerAvailable`, `MarkWorkerUnavailable`, `GetWorker`, `ListWorkers`, `ListWorkerActorAssignments`.
- Tag, template, policy, atespace RPCs: `tag.go:RPCService.CreateTag` (plus Get/List/Update/Delete), `actor_template.go:RPCService.CreateActorTemplate` (plus Get/List/Delete), `egress_policy.go:RPCService.CreateActorEgressPolicy` (plus Get/Update/Delete), `access_policy.go:RPCService.CreateGlobalAccessPolicy` and `RPCService.CreateAtespaceAccessPolicy` (plus Get/Update/Delete for atespace; Get/Update for global), `atespace.go:RPCService.CreateAtespace` (plus Get/List/Delete).
- Worker-reported state: `snapshot_report.go:RPCService.ApplyActorSnapshotState`.
- Orchestration: `workflow.go:ActorWorkflow` (lifecycle in `workflow_resume.go:ResumeActor`, `workflow_suspend.go:SuspendActor`, `workflow_revert.go:RevertActor`, `workflow_delete.go:DeleteActor`, `workflow_worker_crash.go:CrashActorReportedByWorker`, `workflow_tag.go:TagActorSnapshot`, `workflow_template_delete.go:DeleteActorTemplate`), `workflow.go:WorkerWorkflow` (`workflow_reconcile_assignments.go:ReconcileAssignments`, `workflow_worker_delete.go:DeleteWorker`).
- Reconcilers: `template_reconciler.go:ActorTemplateReconciler`, `worker_assignment_reconciler.go:WorkerAssignmentReconciler`.
- Atelet connection: `dialer.go:AteletDialer`, `dialer.go:NewAteletDialer`.
- Metrics: `metrics.go:Instruments`. Volume plugins: `service.go:VolumePluginRegistry` and `service.go:RPCService.GetPlugin`.

## Invariants
- RPC methods stay light and delegate business logic to `ServiceImpl` (`service.go:RPCService` doc).
- `service.go:serviceStore` enumerates the exact storage methods the control API needs and nothing more.
- Every store write is guarded by uid and version preconditions; blind writes are rejected by the store (`store.go:ErrPreconditionRequired`).
- `service.go:NewRPCService` documents that `objectStorePlugin` may be nil only in tests; ate-api always builds one, and nil leaves external snapshots in place.
- `service.go:NewRPCService` copies `actorJWTIssuer` verbatim into the `iss` claim of every minted actor JWT.

## Side Effects
- Persists all resources through `store.Interface` (`service.go:RPCService.persistence`, backed by `./store/atepg`).
- Dials atelets over mTLS through `dialer.go:AteletDialer` for suspend, resume, and snapshot flows.
- Calls the object-store control plugin `objectstore.ControlProviderClient` to copy and release snapshots (`service.go:RPCService`).
- Signs actor JWTs with `internal/localjwtauthority.Pool` (`service.go:RPCService.actorIDJWTPool`).
- Reads cluster state via Kubernetes listers (`service.go:RPCService.csiDriverConfigLister`, `storageClassLister`).
- Emits OTel metrics via `metrics.go:Instruments`.

## Verification
- Run: `go test ./cmd/ateapi/internal/controlapi/...`
