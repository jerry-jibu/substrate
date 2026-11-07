# roottest

## Overview

Gates tests that need root privileges (mounts, mknod, trusted.* xattrs).
Importing the package is also the marker `hack/run-root-tests.sh` uses to
discover which packages to rerun as root.

## Interface

- `roottest.go:Require` - skips the test via `tb.Skipf` unless `os.Geteuid() == 0`; the reason names the privileged operation.
- Subdirectories: none.

## Invariants

- Unprivileged runs skip, never fail; the skip message points at `hack/run-root-tests.sh` (`roottest.go:Require`).

## Side Effects

- ["None"]

## Verification

- Run: `go test ./internal/roottest/...` (root-dependent tests need `hack/run-root-tests.sh`)
