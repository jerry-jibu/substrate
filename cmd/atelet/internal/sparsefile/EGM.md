# sparsefile

## Overview

Copies files while preserving holes, and measures the disk space a file's
data takes. The biggest thing atelet copies is a guest memory image, mostly
unallocated; a plain `io.Copy` would inflate it to its full logical size and
destroy the sparseness later stages rely on.

## Interface

- `sparsefile.go:CopyFile` - copies `src` to `dst` preserving holes where it can; returns logical bytes copied; `dst` created like `os.Create`.
- `sparsefile.go:Copy` - copies between open handles for callers that control how endpoints are opened; clobbers both offsets, closes neither.
- `sparsefile.go:AllocatedBytes` - disk space allocated to a file (`st_blocks * 512`), not apparent size.
- `copyrange_linux.go:kernelCopyRange` (unexported) - `copy_file_range` wrapper, Linux only; `copyrange_other.go` stubs it as unsupported off Linux so callers copy through userspace.

## Invariants

- Sparse copy is located with `SEEK_DATA`/`SEEK_HOLE`; a filesystem that cannot report holes falls back to a dense copy with `dst` untouched (probe before writing) (`sparsefile.go:copySparse`).
- The destination of `Copy` must be empty (freshly created): holes are never written, so pre-existing bytes would show through and silently corrupt the copy; a non-empty destination is rejected up front (`sparsefile.go:Copy`).
- Every seek iteration must advance the offset; a non-advancing hole offset is an error, not a spin (`sparsefile.go:copySparse`).
- Kernel range copy (`copy_file_range`, capped at `maxKernelCopy` 1 GiB per call) degrades to a userspace copy on `errKernelCopyUnsupported` (e.g. EXDEV across filesystems), redoing the failed chunk (`sparsefile.go:copySparse`, `copyrange_linux.go:kernelCopyRange`).
- A file that is one big hole (SEEK_DATA returns ENXIO) is reproduced as a truncate to the logical size (`sparsefile.go:copySparse`).

## Side Effects

- Filesystem: reads source files, creates/truncates/ writes destination files (`sparsefile.go:CopyFile`, `sparsefile.go:copySparse`).

## Verification

- Run: `go test ./cmd/atelet/internal/sparsefile/...`
