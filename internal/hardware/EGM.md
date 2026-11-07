# hardware

## Overview

Probes host hardware compatibility attributes and evaluates sandbox runtime
compatibility between workers and snapshots. Attributes include architecture,
CPU features (a gVisor-derived bitmask), CPU vendor, and CPU model.

## Interface

- Attribute key constants: `SchemaVersionV1`, `AttrArchitecture`, `AttrSandboxBundleHash`, `AttrCPUFeatures`, `AttrCPUVendor`, `AttrCPUModel` (`hardware.go:SchemaVersionV1`).
- `ProbeHost() *ateletpb.VersionedSandboxCompat` (`hardware.go:ProbeHost`): minimal v1 compat with the host architecture, for tests and benchmarks.
- `ProbeCPUIdentity() (vendor, model string, err error)` (`cpufeatures.go:ProbeCPUIdentity`): reads `/proc/cpuinfo` for the microVM vendor/model attributes.
- `EncodeCPUFeatures(arch string, features []string) (string, error)` (`cpufeatures.go:EncodeCPUFeatures`): encodes `runsc cpu-features` names as the canonical hex `cpu_features` mask.
- `DecodeCPUFeatures(arch, value string) ([]string, error)` (`cpufeatures.go:DecodeCPUFeatures`): decodes a mask back to feature names in bit order.
- `Requirement` / `NewRequirement(snap *ateapi.ActorSandboxRuntime) Requirement` (`hardware.go:NewRequirement`): a snapshot's restore requirement prepared for many comparisons.
- `Requirement` methods: `Runtime`, `Matches(rt)`, `MatchingRuntime(status)`, `WorkerMatches(status)`, `WorkerRuntimeFor(status)`, `RuntimeFor(status)` (`hardware.go:Matches`).
- Package-level one-shot helpers wrapping `NewRequirement`: `Matches`, `MatchesCompat`, `MatchingRuntime`, `WorkerMatches`, `RuntimeFor` (`hardware.go:Matches`).
- `ColdBootRuntime(status) *ateapi.ActorSandboxRuntime` (`hardware.go:ColdBootRuntime`) and `SandboxClass(status) string` (`hardware.go:SandboxClass`).

## Invariants

- A requirement of nothing (nil snapshot) is met by any runtime and worker (`hardware.go:Matches`).
- Compat matching requires the same non-empty schema version and every snapshot attribute satisfied by the worker (`hardware.go:matchesCompat`).
- `AttrCPUFeatures` is satisfied when the snapshot's bitmask is a subset of the worker's; every other attribute must be equal; worker-only keys are ignored so a new key under the same schema version does not invalidate older snapshots (`hardware.go:attributeSatisfied`, `hardware.go:matchesCompat`).
- The `cpu_features` encoding is canonical: lowercase hex, no prefix, no leading zeros; equal sets encode to equal strings (`cpufeatures.go:featureMask.String`).
- An unknown feature name is an encode error rather than a dropped bit, and a set bit with no name is a decode error (`cpufeatures.go:EncodeCPUFeatures`, `cpufeatures.go:DecodeCPUFeatures`).
- Only amd64 and arm64 have feature tables (`cpufeatures.go:cpuFeatureTables`); a table bit outside the 6-word mask is a construction panic (`cpufeatures.go:newFeatureTable`).
- A snapshot `cpu_features` value that is not a canonical mask matches no worker (`hardware.go:compatRequirement`).

## Side Effects

- `ProbeCPUIdentity` reads `/proc/cpuinfo` from disk (`cpufeatures.go:ProbeCPUIdentity`).

## Verification

- Run: `go test ./internal/hardware/...`
