# atelet

## Overview

Shared constants and definitions used across Substrate components related to
atelet, the node-local agent. The atelet binary's implementation lives in
subdirectories of this path; no child contract files exist on disk yet, so none
are linked here.

## Interface

- `DefaultPort = 21998` (`port.go:DefaultPort`): the port atelet's gRPC server listens on; also its hostPort, chosen to avoid common node ports, the NodePort range, and the ephemeral range.

## Invariants

- `DefaultPort` serves both the gRPC server and the hostPort, so it must stay outside the NodePort and ephemeral ranges (`port.go:DefaultPort` doc).

## Side Effects

- None.

## Verification

- Run: `go test ./internal/atelet/...`
