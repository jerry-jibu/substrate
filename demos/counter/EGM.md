# counter

## Overview

The primary Substrate demo. `counter.go` is an HTTP server that increments
counters in memory, on a durable volume, and on the root filesystem, so a
snapshot's behavior across suspend/resume is observable: memory and rootfs
counters survive with MEMORY/ROOTFS fidelity, the durable volume always
persists. Install templates provision a 3-replica gVisor WorkerPool.

## Interface

- `main` (`counter.go:68`): HTTP server on port 80 with optional
  `--extra-port` (HTTP) and `--tcp-port` (TCP echo) listeners
  (`counter.go:74`-`:75`).
- Flags (`counter.go:70`-`:75`): `--file-counter-directory` (default
  `/home/counter`), `--second-file-counter-directory`,
  `--rootfs-counter-file`, `--validate-existing-file-path`, `--extra-port`,
  `--tcp-port`.
- Endpoints: `/` returns all counters (`counter.go:92`), `/readyz` gates on
  completed initialization and is the wakeup-probe endpoint
  (`counter.go:128`), `/set-sigterm-sleep` tunes SIGTERM handling
  (`counter.go:137`).
- `incrementFileCounter` (`counter.go:50`), `writeRandomFile`
  (`counter.go:228`) writing 1 MiB of random data to
  `/random-content-file`, and `hashRandomFile` (`counter.go:243`) logging
  its SHA-256 every 10 seconds (`counter.go:221`).
- Install templates: `counter.yaml.tmpl` (Namespace `ate-demo-counter` +
  `WorkerPool/counter`, 3 replicas, gvisor class, `counter.yaml.tmpl:19`-`:60`)
  and `counter-template.yaml.tmpl` (the ActorTemplate, created via
  `kubectl ate`, `counter.yaml.tmpl:15`-`:17`); microVM variants in
  `counter-microvm*.yaml.tmpl`.

## Invariants

- `/readyz` returns 200 only after the random file is written; after a
  checkpoint+restore the ready flag is part of the snapshot, so it answers
  200 immediately on resume (`counter.go:124`-`:135`).
- The rootfs counter distinguishes snapshot fidelities: ROOTFS and MEMORY
  preserve it, VOLUMES resets it (`counter.go:72`).
- File counter writes serialize through `fileMutex` (`counter.go:46`,
  `:50`-`:52`).

## Side Effects

- Writes counter files to the durable volume directories and the root
  filesystem (`counter.go:94`-`:95`, `:114`) and 1 MiB of random data to
  `/random-content-file` (`counter.go:229`).
- Delays exit on SIGTERM for a configurable duration before `os.Exit(0)`
  (`counter.go:79`-`:87`).

## Verification

- Run: `go test ./demos/...`
