# testenv

## Overview

Starts the envtest (kubebuilder) control plane shared by the repo's
apiserver-backed test packages: a kube-apiserver plus etcd pair with the
repo CRDs from `manifests/ate-install/generated` installed.

## Interface

- `testenv.go:Start` - call from TestMain; returns a `*rest.Config` and a stop func. Under `-short` it exits the test binary with success, so `go test -short ./...` works offline.
- `testenv.go:kubeVersion` - pins the control plane to the newest installed `1.36.x` patch (`testenv.go:binaryAssetsDir` via `hack/run-tool.sh setup-envtest use`).
- `testenv.go:lockEnvtestSetup` - cross-process `syscall.Flock` on `<TMPDIR>/substrate-setup-envtest.lock` serializing the first binary download across the parallel test binaries.
- Subdirectories: none.

## Invariants

- The envtest minor version must track `k8s.io/*` in go.mod (`testenv.go:kubeVersion` doc).
- The first `setup-envtest` download is serialized; later callers hit a warm cache (`testenv.go:lockEnvtestSetup` doc).

## Side Effects

- Runs kube-apiserver and etcd processes (`testenv.go:Start`); executes `setup-envtest`, which may download binaries on a cold cache; creates and locks a file under `os.TempDir()`.

## Verification

- Run: `go test ./internal/testenv/...` (its consumers run the control plane from their TestMain)
