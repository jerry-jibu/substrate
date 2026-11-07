# kubectl-ate/internal/cmd

## Overview
Cobra command tree for the `kubectl ate` plugin: verb groups (get/create/update/delete/resume/suspend/logs/top) over actor, worker, actor-template, atespace, tag, egress-policy, and access-policy resources, plus admin key management, revert, and a local proxy.

## Interface
- `root.go:Execute` - entry point called by `cmd/kubectl-ate/main.go`.
- `root.go:rootCmd` - root command with persistent flags: `--kubeconfig`, `--context`, `--endpoint` (manual gRPC target; otherwise port-forwards), `--token-file` (or `-` for stdin), `--output` (table/json/yaml), `--trace`, `--verbose` (repeatable), `--preview`.
- `verbs.go` var block - verb group commands: `get`, `create`, `update`, `delete`, `resume`, `suspend`, `logs`, `top`.
- Resource commands: `actor.go` (`actor [actor-name ...]` plus six `actor <actor-name>` subcommands: create, update, delete, resume, suspend, logs, lines 67-276), `revert_actor.go` (`actor <actor-name>` under revert), `actortemplate.go`, `atespace.go`, `tag.go`, `worker.go`, `egress_policy.go`, `access_policy.go`.
- `admin.go` - `admin make-ca-pool`, `make-jwt-pool`, `list-jwt-keys`, `add-jwt-key`, `activate-jwt-key`, `remove-jwt-key` (lines 46-215).
- `proxy.go:proxyCmd` - `proxy actor <actor-name> [[local-port:]remote-port ...]` serving a local proxy to a resource.

## Invariants
- `root.go:rootCmd` PersistentPreRunE rejects `-o` values other than table, json, or yaml.
- `--preview` sets the preview header on the outgoing gRPC context (`root.go:rootCmd` PersistentPreRunE).
- `root.go:executeRoot` prints usage only for usage errors; runtime errors print the error alone (SilenceUsage/SilenceErrors are set on the root).
- `root.go:executeRoot` turns unknown args to group commands into subcommand errors; `root.go` init registers `--version` without a `-v` shorthand so `-v` stays free for `--verbose`.

## Side Effects
- Connects to ate-api over gRPC, port-forwarding through Kubernetes when `--endpoint` is omitted (`root.go` flag description).
- Manifest-based commands (`-f <manifest>`) read files; `proxy.go` opens local listeners for port mappings.
- `admin.go` writes CA/JWT pool Secrets in the target cluster.

## Verification
- Run: `go test ./cmd/kubectl-ate/internal/cmd/...`
