# cdi

## Overview
Minimal reader for the CDI (Container Device Interface) spec JSON: the devices and the container edits each device applies, used when injecting accelerator devices into worker sandboxes.

## Interface
- `cdi.go:Spec` - the consumed part of a CDI spec: devices and the edits applied to every container.
- `(*Spec).EditsFor(devices)` (`cdi.go:EditsFor`) - returns the merged container edits for the given devices; the consumption entry point used by `cmd/ateworker-gvisor/internal/cdiinject/inject.go`.
- `cdi.go:Device`, `cdi.go:Edits`, `cdi.go:Dev`, `cdi.go:Mount`, `cdi.go:Hook` - spec element types.
- `cdi.go:Parse` - reads a CDI spec in JSON form.

## Invariants
- Generators name the same hardware several ways (per-index, per-UUID, and an "all" device repeating every node), so applying every device in a spec injects each node several times over; consumers must deduplicate (`cdi.go:Device`).
- Major/Minor on a `Dev` are often absent: CDI leaves resolving them to the OCI runtime, which stats the host (`cdi.go:Dev`).

## Side Effects
- None. Pure JSON parsing.

## Verification
- Run: `go test ./internal/cdi/...`
