# ate-setup/internal/demos/counter

## Overview

Installs the counter demo: a counter actor exercising snapshot, resume, and
atenet ingress, optionally with an external volume attached so the CSI path is
covered end to end (`counter.go` package doc).

## Interface

- `demo` (`counter.go`) - embeds `demos.Substrate` and registers as `demo-counter`, with pool manifest `demos/counter/counter.yaml.tmpl` and template `counter` in atespace `ate-demo-counter`.
- `Settings` (`counter.go`) - `demo.counter.withExternalVolume` (flag `--with-external-volume`) and `demo.counter.storageClass` (flag `--storage-class`, default `standard`).
- `renderValues` / `externalVolumeValues` (`counter.go`) - supply the `VALIDATE_EXISTING_FILE_PATH_ARG`, `EXTERNAL_VOLUME_MOUNTS`, and `EXTERNAL_VOLUMES` placeholder values when the external-volume flag is set.

## Invariants

- With `--with-external-volume` unset, the three external-volume placeholders are dropped, removing those template lines entirely (`counter.go` `renderValues` with `demos.ExternalVolumePlaceholders`).
- `Commands` is left empty in `Settings`; `demos.Register` fills in the demo's deploy command path (`counter.go` `Settings`).

## Side Effects

- Deploys the `ate-demo-counter` namespace, WorkerPool, and ActorTemplate, and waits for the golden snapshot, through `demos.Substrate.Deploy` (`counter.go` `init` wiring).

## Verification

- Run: `go test ./cmd/ate-setup/internal/demos/counter/...`
