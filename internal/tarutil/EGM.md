# tarutil

## Overview

Archives and restores a directory tree as a tar file, preserving the
metadata a workload's data directory depends on: modes, ownership,
modification times, symlinks, hardlinks, FIFOs, overlay whiteouts, and
`user.*` / `trusted.overlay.*` extended attributes as PAX `SCHILY.xattr`
records. Built for snapshotting durable-dir volumes and rootfs overlay
uppers (see `cmd/ateworker-microvm`); losing a whiteout or an opaque marker
would silently resurrect deleted content after a resume.

## Interface

- `tarutil.go:Create`/`CreateFiltered` - archive a directory to `tarPath`; entry names relative to srcDir; sockets and non-whiteout device nodes are skipped with a warning (`tarutil.go:writeTree`).
- `tarutil.go:CreateWithRoot`/`CreateFilteredWithRoot` - plus a `./` root entry carrying srcDir's own metadata, which Extract applies to dstDir; the root entry is never skipped.
- `tarutil.go:SkipFunc` - omit entries (and subtrees) by relative path.
- `tarutil.go:WriteBehind` (option) - bound how much of the archive may sit dirty in the page cache before write-out starts (`tarutil.go:writeBehind` via `unix.SyncFileRange`, default `tarutil.go:writeBehindChunk` 8 MiB).
- `tarutil.go:Extract` - unpack into an existing dstDir through an `os.Root`; later entries replace earlier ones (standard tar semantics), existing directories are kept when the entry is a directory; directory modes are applied deepest-first after children exist (`tarutil.go:restoreDirMeta`).
- `tarutil.go:isWhiteout` - a 0:0 character device; the only device `writeTree` archives and `extractEntry` restores.
- `tarutil.go:keepXattr` - keeps `user.*` and `tarutil.go:overlayXattrs` (`trusted.overlay.opaque|redirect|impure`); everything else, notably `security.capability`, is dropped.
- Helpers in `owner.go`/`owner_linux.go` (`Lchown`, `Mode`) and `fifo_linux.go` (`createFifo`, `createDevice`).

## Invariants

- Both archive and extract walks are confined by `os.Root`, so a crafted archive or a mid-walk symlink swap cannot reach outside the tree; regular files are opened `O_NOFOLLOW|O_NONBLOCK` and re-verified regular (`tarutil.go:writeTree`, `tarutil.go:openRegular`, `tarutil.go:Extract`).
- Extraction refuses every device node except overlay whiteouts (`tarutil.go:extractEntry` `TypeChar`/`TypeBlock` case).
- Archives are written in lexical order so identical trees archive byte-comparably, keeping snapshot diffs meaningful (`tarutil.go:writeTree` doc).
- No fsync: local snapshots are not trusted across a node restart anyway (`atelet` stamps the boot id), and fsync would wait behind other actors' journal writes (`tarutil.go:create` comment, #3050/#1954).
- Ownership is applied before mode because chown clears setuid/setgid, which the chmod then restores (`tarutil.go:restoreMeta`).
- Xattr writes are addressed through the parent directory fd, never by joining host paths, and `Lsetxattr` does not follow a final symlink (`tarutil.go:restoreOverlayXattrs`).
- Entries naming paths outside the destination (`..`, absolute, symlinked) are refused (`tarutil.go:cleanTarName`).
- Multi-link files archive once and become tar hardlink entries thereafter (`tarutil.go:writeTree` `linked` map).

## Side Effects

- Creates and writes the tar file (`tarutil.go:create`), starts kernel write-out of dirty chunks (`tarutil.go:writeBehind`), and on extract creates/removes files, dirs, symlinks, FIFOs, whiteout devices, and sets xattrs and times under dstDir.

## Verification

- Run: `go test ./internal/tarutil/...`
