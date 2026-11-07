# clientset/versioned/scheme

## Overview
The runtime scheme for the generated clientset, built by `client-gen` (`register.go` and `doc.go` carry `Code generated ... DO NOT EDIT`). Registers the `api/infra/k8s/ate/v1` types so the typed clients can serialize them.

## Interface
- Package-level `Scheme` (`runtime.NewScheme`), `Codecs` (`serializer.NewCodecFactory(Scheme)`), and `ParameterCodec` (`register.go:Scheme`, `register.go:Codecs`, `register.go:ParameterCodec`)
- `AddToScheme` to compose this clientset's types into another scheme (`register.go:AddToScheme`)

## Invariants
- The `init` function registers the group and adds all types with `utilruntime.Must`, so importing the package fails fast on registration errors (`register.go:init`).
- The scheme builder is exactly `atev1.AddToScheme` from `api/infra/k8s/ate/v1` (`register.go:localSchemeBuilder`).

## Side Effects
- Package `init` mutates the shared package-level `Scheme` (`register.go:init`).

## Verification
- Run: `go test ./pkg/client/clientset/versioned/scheme/...` from the repo root.
