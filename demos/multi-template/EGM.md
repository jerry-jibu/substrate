# multi-template

## Overview

Demo showing that two ActorTemplates running two entirely different binaries
(the counter and fspersist) can share a single WorkerPool. The templates are
created through the ate API with `kubectl ate create actor-template`; the
pool and namespace come from `multi-template.yaml.tmpl`.

## Interface

- `multi-template.yaml.tmpl`: install template with the shared WorkerPool
  and namespace for both demo templates.
- `counter-template.yaml.tmpl`: ActorTemplate for the counter binary.
- `fspersist-template.yaml.tmpl`: ActorTemplate for the fspersist binary.
- fspersist: [./fspersist/EGM.md](./fspersist/EGM.md) - the filesystem
  persistence workload; the counter binary itself lives in
  `demos/counter`.

## Invariants

- Both templates select the same shared WorkerPool, which is the
  multiplexing the demo demonstrates (`fspersist.go:19`-`:21` describes the
  pairing with the counter demo).
- fspersist proves filesystem persistence while counter proves memory
  persistence (`fspersist.go:21`-`:27`).

## Side Effects

- Applying the templates creates the namespace, WorkerPool, and two
  ActorTemplates in the target cluster.

## Verification

- Run: `go test ./demos/...`
