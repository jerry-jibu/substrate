# fixtures

## Overview

Workload fixtures for the e2e suites: the introspection probe and the multi-mode testserver, each with the YAML templates (`*.yaml.tmpl`) that deploy them through the e2e harness's manifest rendering. The template-only directories `capabilities`, `credinject`, and `security` hold fixture YAML with no Go code.

## Subdirectories & Boundaries

- [probe](./probe/EGM.md) - the in-actor introspection HTTP server and its deployment templates (sized, parking, multiactor, bootfetch, nonroot variants).
- [testserver](./testserver/EGM.md) - the one-binary helper pod for egress suites (grpc, http, egressprobe, websocket, kill, procstatus subcommands) and its templates.
- `capabilities/`, `credinject/`, `security/` - YAML fixture templates only (no Go code, no contract file).
