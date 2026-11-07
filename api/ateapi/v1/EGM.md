# ateapi/v1

## Overview

The `substrate.ateapi.v1` proto package is the public Control API of Agent
Substrate, served by `ate-api-server`. `ateapi.proto` is the source of truth;
`ateapi.pb.go` and `ateapi_grpc.pb.go` are generated from it, and `source.go`
embeds the proto text so tools such as `apitool` can parse it without reading
files. Go package: `go.ate.dev/substrate/api/ateapi/v1` (package `ateapi`).

## Interface

- `service Control` (`ateapi.proto:25`): the primary RPC interface. Actor CRUD and lifecycle (`GetActor`, `CreateActor`, `UpdateActor`, `SuspendActor`, `ResumeActor`, `RevertActor`, `DeleteActor`, `ListActors`), per-actor nested `EgressPolicy` CRUD (`GetActorEgressPolicy` ... `DeleteActorEgressPolicy`), `MintActorJwt`, Tag CRUD (`CreateTag`, `GetTag`, `ListTags`, `UpdateTag`, `DeleteTag`), Worker registry and lifecycle (`ListWorkers`, `GetWorker`, `CreateWorker`, `UpdateWorker`, `DeleteWorker`, `DrainWorker`, `MarkWorkerAvailable`, `MarkWorkerUnavailable`, `ListWorkerActorAssignments`), Atespace CRUD, ActorTemplate CRUD, and global/atespace `AccessPolicy` singletons.
- `service WorkerService` (`ateapi.proto:2641`): the RPCs atelet calls with its own client certificate: `AcquireWorkerEpoch`, `RegisterWorker`, `SetWorkerSandboxRuntimes`, `MintWorkerActorCertificate`, `MintWorkerActorJwt`, `RequestActorSuspend`, `ReportActorSnapshotState`, `ReportActorCrashed`.
- Resource messages with `(resource)` options: `Actor` (atespace scope), `EgressPolicy` (singleton under `Actor`), `Tag`, `Atespace` (global scope), `ActorTemplate`, `AccessPolicy` (singleton under Global and Atespace) (`ateapi.proto:538`, `:588`, `:1085`, `:1121`, `:1153`, `:3085`).
- Worker model: `Worker`, `WorkerStatus`, `WorkerState`, `WorkerSandboxClass`, `WorkerSandboxRuntime`, `ActorSandboxRuntime`, `VersionedSandboxCompat`, `WorkerResources`, `ActorAssignment` (`ateapi.proto:2281`-`:2641`).
- Snapshot model: `Snapshot`, `SnapshotStorage`, `ObjectSnapshot`, enums `SnapshotOwner`, `SnapshotStorageState`, `SnapshotFidelity`, `DurabilityMode` (`ateapi.proto:266`-`:383`, `:1829`).
- Egress model: `EgressRule`, `HttpRule`, `HttpsRule`, `TlsPassthroughRule`, `Ports`, `AllPorts`, `HttpRuleEffects`, `CredentialHeader`, `ActorJwtSource` (`ateapi.proto:650`-`:877`).
- Template model: `ResourceRequirements`, `SnapshotConfig`, `GoldenSnapshotConfig`, `Container`, `Volume`, `VolumeMount`, `EnvVar`, `WakeupProbe`, `HttpGetAction`, `SecurityContext`, `Capabilities`, `SystemInfoVolumeSource` family (`ateapi.proto:1220`-`:1681`).
- Generated Go server/client stubs: `ControlServer`/`ControlClient`, `WorkerServiceServer`/`WorkerServiceClient`, `RegisterControlServer`, `RegisterWorkerServiceServer` (`ateapi_grpc.pb.go:633`, `:85`, `:2056`, `:1858`, `:892`, `:2200`), and all message structs in `ateapi.pb.go`.
- `Source` (`source.go:23`): `go:embed ateapi.proto` exposing the proto text to importers.

## Invariants

- Every served RPC carries a `(method).resource` option binding it to a resource (`ateapi.proto:27`-`:262`); `ResourceAnnotation` and `MethodAnnotation` (`ateapi.proto:3227`, `:3256`) define the annotation schema.
- `ResourceMetadata.uid` and `.version` are ignored on create and used as preconditions on update; `version` increases on every mutation (`ateapi.proto:431`-`:451`).
- Status fields (`Actor.status`, `Tag.status`, `ActorTemplate.status`) are server-managed and ignored on input (`ateapi.proto:578`, `:1097`, `:1216`).
- `EgressPolicy` and `AccessPolicy` are singletons whose `metadata.name` must be `default` (`ateapi.proto:588`, `:3085`).
- Egress is default-deny: traffic is refused when no rule matches, and rules with the same pattern on the same port are rejected at admission (`ateapi.proto:604`-`:624`).
- Container and volume images must be pinned by digest (`Container.image`, `ImageVolumeSource.reference`, `ateapi.proto:1331`, `:1533`).
- Worker registration is epoch-guarded: `RegisterWorker` requires an epoch `AcquireWorkerEpoch` handed out, higher than `registered_epoch`, and immutable once registered (`ateapi.proto:2652`-`:2664`).
- `AcquireWorkerEpoch` is not idempotent; `ReportActorSnapshotState` is safe to repeat; `RequestActorSuspend` is refused NotFound for an Actor the caller no longer hosts (`ateapi.proto:2649`, `:2730`, `:2717`).
- The `+k8s:*` annotations in the proto drive generated Kubernetes-style validation bounds (e.g. `Snapshot.generation` minimum 1, `ateapi.proto:316`-`:324`).

## Side Effects

- `RegisterControlServer` and `RegisterWorkerServiceServer` attach handlers to a `grpc.ServiceRegistrar` (`ateapi_grpc.pb.go:892`, `:2200`); the package itself is otherwise pure type definitions.
- `source.go` embeds `ateapi.proto` into the compiled binary via `go:embed` (`source.go:22`).

## Verification

- Run: `go test ./api/...`
