# workersync

## Overview
Syncs the set of worker Pods in the cluster into the Worker registry of ate-api-server. `syncer.go:WorkerPoolSyncer` watches worker Pods and per-pool informers, then drives the ateapi Control API so the registry matches reality, including deleting Pods that reached a terminal phase or outlive their pool.

## Interface
- `syncer.go:WorkerPoolSyncer`, built with `syncer.go:NewWorkerPoolSyncer(client ateapi.ControlClient, pods corev1client.PodsGetter, apps appsv1client.AppsV1Interface, workerInformer, workerPoolInformer cache.SharedIndexInformer, workerCount int)`, started with `syncer.go:WorkerPoolSyncer.Start(ctx)`.
- `informer.go:WorkerPodInformer(kc kubernetes.Interface)` builds the shared worker-Pod informer.

## Invariants
- Distinguishing a terminal-phase Pod from a Pod whose pool shrank requires reading the Pod's ReplicaSet, which is why `gen.go` in `internal/controllers` holds the `replicasets;verbs=get` RBAC marker (`gen.go` doc in the sibling controllers package).

## Side Effects
- Lists and watches worker Pods; deletes worker Pods that are terminal or orphaned by their pool.
- Calls the ateapi Control API (create/update/delete Workers) through `ateapi.ControlClient`.

## Verification
- Run: `go test ./cmd/atecontroller/internal/workersync/...`
