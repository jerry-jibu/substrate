# ate-setup/internal/render

## Overview

Expands the demos' `*.yaml.tmpl` files. The install scripts drove these through
sed with two distinct behaviors, substitution and line deletion, which this
package keeps (`render.go` package doc).

## Interface

- `Template` (`render.go`) - render a `*.yaml.tmpl` file: substitute `values`, and drop every line mentioning a placeholder named in `drop`.
- `Expand` (`render.go`) - apply the substitution and line-deletion rules to a template body.

## Invariants

- Any placeholder named in `drop` causes every line mentioning it to be removed, so a demo opts lines in by supplying a value and out by dropping it (`render.go` `Template`).

## Side Effects

- Reads template files from the repository (`render.go` `Template`).

## Verification

- Run: `go test ./cmd/ate-setup/internal/render/...`
