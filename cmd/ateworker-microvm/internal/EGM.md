# internal

## Overview
Packages private to the microVM ateworker binary: cloud-hypervisor support, sandbox staging derived from kata-containers, child process supervision, and agent stats conversion.

## Subdirectories & Boundaries
- `ch`: `./ch/EGM.md` - cloud-hypervisor VM config, API client, overlay merge, and VMM launch.
- `kata`: `./kata/EGM.md` - sandbox paths, agent client, virtiofsd and overlay staging, spec conversion.
- `child`: `./child/EGM.md` - supervised child processes.
- `agentstats`: `./agentstats/EGM.md` - converts agent cgroup stats for reporting.
- `third_party/kata/agentpb`: vendored kata agent protobuf types (generated code, no EGM contract).
