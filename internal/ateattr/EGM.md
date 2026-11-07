# ateattr

## Overview
The single source of attribute-key spellings and bounded label values shared by metrics, logs, and traces. Producers build identity and dimension attributes here so components cannot drift on spellings, and unmapped values collapse to a bounded `unknown` rather than widening a label set.

## Interface
- `ateattr.go:ReservedNamespace` - `ate.`, the namespace producers strip from untrusted merged fields.
- `ateattr.go:TemplateUnknown`, `ateattr.go:NormalizeTemplateDimension` - bounded fallback for template dimensions.
- `ateattr.go:ActorStateValue` - maps committed `ateapi.ActorState` onto its label value; `ActorStateKey` and its values are log-only.
- `ateattr.go:ErrorTypeKey` - OTel registry attribute reused verbatim; absence means success.
- `ateattr.go:AllOperations`, `ateattr.go:NormalizeOperationName` - bounded lifecycle operations; unlisted or empty maps to `OperationUnknown`.
- `ateattr.go:SnapshotFidelityValue` - wire enum to label value, shared by ateapi and atelet.
- `ateattr.go:NormalizeSandboxClass`, `ateattr.go:SandboxClassAttribute`, `ateattr.go:SandboxClassUnknown` - bounded sandbox-class label.
- `ateattr.go:WorkerPoolAttributes` - namespace/name pair for worker-pool dimensions.
- `ateattr.go:ActorRefAttributes`, `ateattr.go:ActorAttributes` - span attributes for resolved and unresolved actors.
- `ateattr.go:ActorLogLabels`, `ateattr.go:ActorLogAttrs`, `ateattr.go:ActorRefLogAttrs` - actor identity for log records.
- `ateattr.go:ActorMetricAttributes` - metric labels for an actor.
- `ateattr.go:OTLPRelayKey`, `ateattr.go:SandboxOOMKillsKey`, `ateattr.go:SnapshotUploadOverdueBandKey` - special-purpose attribute keys.

## Invariants
- Actor identity is never a metric label: `ActorStateKey` is log-only because it sits beside actor identity, and `ActorMetricAttributes` omits high-cardinality attributes (atespace, actor name, actor uid) (`ateattr.go:ActorStateKey`, `ateattr.go:ActorMetricAttributes`).
- Unrecognized enum values report as `unknown` rather than their stringified form, so no wire value can widen the label set (`ateattr.go:SnapshotFidelityValue`).
- `WorkerPoolAttributes` emits the namespace/name pair or neither, so same-named pools from different namespaces cannot merge (`ateattr.go:WorkerPoolAttributes`).
- `ActorLogAttrs` must agree with `ActorLogLabels` key for key, or component records and actor output split into two streams (`ateattr.go:ActorLogAttrs`).
- An empty container name omits the key rather than emitting it empty (`ateattr.go:ActorLogLabels`).
- The worker-pool pair is omitted while an actor holds no assignment (`ateattr.go:ActorMetricAttributes`).

## Side Effects
- None. Pure attribute construction.

## Verification
- Run: `go test ./internal/ateattr/...`
