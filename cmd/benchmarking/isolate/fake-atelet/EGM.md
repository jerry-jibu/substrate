# fake-atelet

## Overview

Stands in for atelet in control-plane benchmarks. Runs in atelet's
DaemonSet (installed by `ate-setup --fake-data-plane`), so ate-api-server
dials it as the node's atelet. Answers every Atelet call with success after
a delay, without running or saving any workload, and relays fake-workersync's
capacity reports for the fake Workers on its node, as atelet relays worker's.
Actors it "runs" do not exist.

## Interface

- `main.go:main` - flags: `--port` (default `atelet.DefaultPort`), `--relay-port` (default 8086), `--relay-allowed-spiffe-id` (default ate-system/fake-workersync), `--health-listen-addr`, `--grpc-server-cred-bundle`, `--client-ca-certs`, `--ateapi-address`, `--ateapi-ca-file`, `--ateapi-server-name`, `--delay` and per-call `--delay-run`/`--delay-restore`/`--delay-checkpoint`/`--delay-promote-snapshot`/`--delay-terminate` (negative means use `--delay`), `--mint-actor-certificate`, `--version`, `--log-level`, `--preview`.
- `main.go:serverTLSConfig` - pod-identity certificate as serving cert; client certs required and verified against `--client-ca-certs`, re-read at each handshake.
- `herder.go:herder` - implements `ateletpb.AteletServer`: `herder.go:herder.Run`, `herder.go:herder.Restore`, `herder.go:herder.Checkpoint`, `herder.go:herder.PromoteSnapshot`, `herder.go:herder.Terminate`, each succeeding after its `herder.go:delays` value.
- `relay.go:relay` - serves `ateapi.WorkerService` (`AcquireWorkerEpoch`, `RegisterWorker`, `SetWorkerSandboxRuntimes`); forwards to ate-api-server unchanged, every other WorkerService call Unimplemented.
- `relay.go:relayTLSConfig` - adds a SPIFFE ID check (`fakeworker.VerifyPeerID`) on top of the herder TLS config.
- `epochs.go:workerEpochs` - in-memory map of each Worker's registered epoch, reported from Run and Restore.
- `config.go:applyTuningEnv` - sets each `config.go:tuningFlags` flag left unset on the command line from `FAKE_ATELET_<FLAG>` environment variables.
- `config.go:resolveDelays` - applies per-call overrides over the default; rejects a negative `--delay`.
- Subdirectories: none.

## Invariants

- Never run on a cluster that serves real actors; Atelet calls succeed without running any workload (`main.go:main` doc comment).
- Each port admits only its own caller: ate-api-server on the herder's port, fake-workersync (SPIFFE ID from `--relay-allowed-spiffe-id`) on the relay port (`main.go:main`, `relay.go:relayTLSConfig`).
- A placeholder object `herder.go:placeholderObject` (`fake-dataplane-placeholder`) is written under every snapshot URI before the call returns, because ate-api-server refuses to copy an empty snapshot (`herder.go:writePlaceholder`).
- The mint on Run/Restore that name an egress gateway happens before the delay and its failure fails the call, as tunneled egress fails closed in worker (`herder.go:mintActorCertificate`).
- `workerEpochs` is memory-only: after a fake-atelet restart, resumes on not-replaced Workers fail with Unavailable until fake-workersync replaces them (`epochs.go:workerEpochs` doc comment).
- A command-line flag wins over its `FAKE_ATELET_*` environment variable (`config.go:applyTuningEnv`).
- No Kubernetes client: the ate-api-server target is resolved through DNS only (`main.go:main`).

## Side Effects

- Serves two gRPC servers (Atelet on `--port`, WorkerService relay on `--relay-port`) and `/healthz`+`/readyz` (always 200) on `--health-listen-addr` (`main.go:main`, `main.go:healthHandler`).
- Dials ate-api-server with mTLS (`ateapiauth.DialOptions`) and calls `WorkerService` RPCs including `herder.go:actorCertMinter` `MintWorkerActorCertificate` (`main.go:main`, `herder.go:herder.Run`).
- Writes placeholder objects to object storage; backend picked from `ATE_STORAGE_BACKEND` (`s3` with optional `AWS_S3_USE_PATH_STYLE`, else GCS) (`main.go:newObjectStorage`, `herder.go:writePlaceholder`).

## Verification

- Run: `go test ./cmd/benchmarking/isolate/fake-atelet/...`
