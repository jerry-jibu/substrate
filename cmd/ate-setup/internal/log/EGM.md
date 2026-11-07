# ate-setup/internal/log

## Overview

Prints install progress, preserving the `[step]: name` output the shell
installer produced (`log.go` package doc).

## Interface

- `Step`, `Stepf` (`log.go`) - announce a named install step.
- `Infof`, `Warnf` (`log.go`) - unadorned progress lines; warnings go to stderr.
- `Elapsed` (`log.go`) - print how long an operation took, as `(label took 1.234s)`.
- `Writer`, `SetOutput` (`log.go`) - destination for progress output, and its redirection for tests.

## Invariants

- `Step` takes a plain string rather than a format string, so a step name containing `%` cannot corrupt output (`log.go` `Step`).

## Side Effects

- Writes to the progress writer and stderr (`log.go`).

## Verification

- Run: `go test ./cmd/ate-setup/internal/log/...`
