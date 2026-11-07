# internal

## Overview

Packages internal to the `apitool` tool: the command tree and the API
model, lint rules, exemptions, examples, Kubernetes model, and website
renderer it drives.

## Subdirectories & Boundaries

- cmd: [./cmd/EGM.md](./cmd/EGM.md) - the `apitool validate` and `generate-docs` commands.
- docsite: [./docsite/EGM.md](./docsite/EGM.md) - website API reference renderer.
- examples: [./examples/EGM.md](./examples/EGM.md) - embedded YAML examples of every resource.
- exemption: [./exemption/EGM.md](./exemption/EGM.md) - exempted findings persisted as sorted JSON.
- k8smodel: [./k8smodel/EGM.md](./k8smodel/EGM.md) - CRD-derived Kubernetes API model.
- lint: [./lint/EGM.md](./lint/EGM.md) - API style rules over the model.
- model: [./model/EGM.md](./model/EGM.md) - proto compilation and API data model.
- validate: [./validate/EGM.md](./validate/EGM.md) - runs the rules and reconciles with exemptions.
