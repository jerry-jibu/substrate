# ate-setup/internal

## Overview

Internal packages of the ate-setup installer: the command tree, configuration
resolution, image building and resolution, cluster operations, manifest
rendering, the bundled demos, and the install steps the shell installer once
implemented as bash functions.

## Subdirectories & Boundaries

- [cmd](./cmd/EGM.md) - the cobra command tree (`deploy`, `delete`, `setup`, `publish`, `create`, `demo`, `benchmarks`).
- [config](./config/EGM.md) - settings registry and layered resolution into a typed `Config`, plus run records.
- [demos](./demos/EGM.md) - the demo contract and registry; one subpackage per bundled demo.
- [images](./images/EGM.md) - component image list, `ko://` conventions, Dockerfile builds, prebuilt rewrites, release checks.
- [ko](./ko/EGM.md) - ko subprocess wrapper for building and publishing images.
- [kube](./kube/EGM.md) - cluster operations replacing kubectl invocations.
- [kustomize](./kustomize/EGM.md) - renders the overlays under `manifests/ate-install`.
- [log](./log/EGM.md) - `[step]`-style install progress output.
- [render](./render/EGM.md) - expands the demos' `*.yaml.tmpl` files.
- [steps](./steps/EGM.md) - the install and teardown steps driven by the commands.
