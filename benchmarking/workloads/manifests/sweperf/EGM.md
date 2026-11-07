# sweperf

Renders the sweperf ActorTemplates from a task catalog: one
`sweperf-<name>-template.yaml.tmpl` per entry in `sweperf-images.json`,
written into `generated/` and checked in. Each template runs a
[gke-labs/sweperf](https://github.com/gke-labs/sweperf) replay image that the
sweperf locust client drives through suspend/resume cycles.

## Interface

- `generate_sweperf_templates.py:load_catalog` - reads `sweperf-images.json` and rejects entries missing `name`/`image`/`steps`, non-positive steps, images not pinned by digest, and duplicate names.
- `generate_sweperf_templates.py:render` - Jinja-renders `sweperf-template.yaml.j2` per task with `StrictUndefined` and `keep_trailing_newline`, keying output by file name.
- `generate_sweperf_templates.py:check` - reports `generated/` files that are missing, stale, or not in the catalog.
- `generate_sweperf_templates.py:main` - CLI: a task name or glob (`astropy-7336`, `'django*'`, `'*'`), `-o/--out` (default `generated/`), `--catalog`, and `--check` to fail on drift.
- `sweperf-images.json` - the five-task catalog (astropy-7336, django-11099, matplotlib-20826, sphinx-8638, sympy-13647), each image pinned by digest with its step count.
- `sweperf-template.yaml.j2` - the template: protojson-shaped ActorTemplate in `benchmark-workloads`, `python3 /replay.py /trace.json` with `SWEPERF_DISABLE_INTERNAL_SLEEP=1`, 2000 millicores / 4 GiB, `SNAPSHOT_FIDELITY_MEMORY`.
- `generated/` - the checked-in render output; edit the catalog or the `.j2`, not these files.

## Invariants

- Catalog images must be pinned by digest because the sweperf images are rebuilt in place under the same tag (`generate_sweperf_templates.py:load_catalog`, `sweperf-template.yaml.j2` image comment).
- The rendered files are checked in; `--check` fails if they are missing, stale, or extra relative to the catalog (`generate_sweperf_templates.py:main`).
- The cpu limit is fixed across runs because it caps the measured execution time (`sweperf-template.yaml.j2` resources comment).
- The client must run the template with `--sweperf-total-steps` equal to the task's steps; the env var is reference only (`sweperf-template.yaml.j2` SWEPERF_TOTAL_STEPS comment).

## Side Effects

- Writes `sweperf-*-template.yaml.tmpl` files into the output directory (`generate_sweperf_templates.py:main`).

## Verification

- Run: `hack/verify/benchmarking-python.sh` (runs this directory's pytest suite).
