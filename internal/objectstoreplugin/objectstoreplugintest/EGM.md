# objectstoreplugintest

## Overview
Provides object-store plugin clients that call the object-store plugins directly, without a socket, for tests that drive atelet or ate-api-server through the real plugin ("Package objectstoreplugintest", `objectstoreplugintest.go`).

## Interface
- `objectstoreplugintest.go:NodeClient(p *objectstoreplugin.NodePlugin) objectstorev1.NodeProviderClient`.
- `objectstoreplugintest.go:AssetClient(p *objectstoreplugin.AssetPlugin) objectstorev1.AssetProviderClient`.
- `objectstoreplugintest.go:ControlClient(store objectstore.Store) objectstorev1.ControlProviderClient`.

## Invariants
- ["None"]

## Side Effects
- ["None"] beyond those of the wrapped plugins themselves (exercised in-process, not over a socket).

## Verification
- Run: `go test ./internal/objectstoreplugin/objectstoreplugintest/...`
