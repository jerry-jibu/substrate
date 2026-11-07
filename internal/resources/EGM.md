# resources

## Overview

Shared control-plane resource helpers: typed resource refs, snapshot URI
and ownership layout in object storage, SPIFFE identity formats, request
validation, worker capacity arithmetic, actor attribution and transfer
states.

## Interface

- `resourceref.go:ResourceRef[R]` - typed `{Atespace, Name}` refs; aliases `ActorRef`, `ActorTemplateRef`, `TagRef` with constructors (`ActorRefFromObjectRef`, `ActorRefFromActor`, `ActorTemplateRefFrom*`, `TagRefFrom*`) and `ToObjectRef`; `String`/`LogValue` for logs.
- `snapshot.go:StoragePrefix`/`ParseStoragePrefix` - validated `scheme://bucket/path` prefix; rejects a bare bucket.
- `snapshot.go:SnapshotOwner`, `ActorSnapshotOwner`, `TagSnapshotOwner` - keyed on the owner's UID (`snapshot.go:validate` checks kind, atespace, id as resource names); `SnapshotOwner.Prefix` builds `<location>/atespaces/<atespace>/<actors|tags>/<uid>`.
- `snapshot.go:SnapshotURI` - `NewActorSnapshotURI`, `NewTagSnapshotURI`, `ParseSnapshotURI`, accessors `Location`/`Atespace`/`Name`/`Owner`/`OwnedBy`/`OwnerPrefix`/`Prefix`/`ObjectURI`; `snapshot.go:NewSnapshotName` - a UUID name.
- `snapshotfiles.go:ValidateSnapshotFileNames` - snapshot file allowlist with `FSCheckpointSubdir` (`fs`) and `RootfsCheckpointSubdir` (`rootfs`).
- `spiffe.go:ActorSPIFFEID`/`ActorRefFromActorSPIFFEID` - `spiffe://substrate-actor.local/actor/<atespace>/<name>` (`spiffe.go:ActorSPIFFETrustDomain`); `spiffe.go:WorkerForActorSPIFFEID`/`ActorRefFromWorkerForActorSPIFFEID` - the worker-for-actor variant.
- `validate.go:ToGRPCStatusError`/`ToAPIError`/`ToInternalError` - `field.ErrorList` to errors, with bad values redacted when marked debug_redact (`validate.go:redactBadValues`).
- `validate.go:Validate*` family - `ValidateResourceName`/`IsValidResourceName` (pattern `actor.go:ResourceNameRegexPattern`), `ValidateWorkerPodUID`, `ValidateActorDirs`, `ValidateSnapshotFidelity`, `ValidateSandboxClass`, `ValidateRuntimeAssetPath`, `ValidateSandboxAssetsDir`, `ValidateSnapshotLocation`, `ValidateIP`, `ValidateUUID`, `ValidatePinnedImage`, `ValidateAddCapabilities`/`ValidateDropCapabilities` (`validate.go:AllowedAddCapabilities`), `ValidateMountPath`, `ValidateNestedMountPaths`, `ValidateHTTPGetPath`, `ValidateEnvVarName`, `ValidateProjectedPath`, `ValidateStorageVolumeID`, `ValidateVolumeType`, `ValidateCSIMap` (40 KiB cap, `validate.go:csiMapMaxBytes`), `ValidateHostPort`; `validate.go:DeepEqual` - protobuf-aware deep equality.
- `compute.go:CPUMemory`/`Covers` - resource limit construction and coverage checks.
- `worker.go:AddToAllocated`/`SumAllocated`/`Free` - WorkerResources arithmetic over ActorAssignments.
- `networkpolicy.go:NetworkPolicyName` - per-WorkerPool NetworkPolicy name.
- `actorstate.go:TransferStates` - RESUMING, RUNNING, SUSPENDING, SUSPENDED.
- `actorattribution.go:ActorAttribution`/`ActorAttributionFromActor` - telemetry attribution (ref, UID, template).
- `singleton.go:SingletonName` - `"default"`, the name of singleton resources.

## Invariants

- Snapshot ownership is keyed on the owner's UID, so a resource recreated under the same name never inherits its predecessor's objects; deletion is expressed as collecting everything under the owner's own prefix (`snapshot.go:ActorSnapshotOwner`, `snapshot.go:SnapshotOwner` doc).
- Ownership is readable from an object's name via the `actors`/`tags` kind segments (`snapshot.go:actorsOwnerKind`, `snapshot.go:tagsOwnerKind`).
- `TransferStates` is the single list both the mint's host check and the objectstore credential exchange read, so they cannot disagree; no node may hold a snapshot credential in other states (`actorstate.go:TransferStates` doc).
- `ActorAttribution` is not a credential or proof of identity; it is telemetry attribution only (`actorattribution.go` doc).
- SPIFFE IDs parse strictly: scheme, trust domain, no user/query/fragment, exactly `actor|worker-for-actor/<atespace>/<name>` with valid resource names (`spiffe.go:ActorRefFromActorSPIFFEURL`).
- Validation errors redact bad values of debug_redact-marked fields before reaching a log or status (`validate.go:redactBadValues`).

## Side Effects

- ["None"] (pure computation and validation)

## Verification

- Run: `go test ./internal/resources/...`
