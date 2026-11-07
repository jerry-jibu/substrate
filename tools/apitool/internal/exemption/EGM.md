# exemption

## Overview

Records lint findings exempted from failing `apitool validate`, persisted
as sorted JSON in `tools/apitool/exemptions.json`. An exemption's identity
is the exact `(Rule, Subject, Message)` triple, so a new finding cannot be
confused with an exempted one that shares its subject.

## Interface

- `exemption.go:Exemption` - `{rule, subject, message}` JSON object.
- `exemption.go:Load` - reads the file; a missing file is zero exemptions, not an error.
- `exemption.go:Save` - writes sorted, indented JSON (canonical order via `exemption.go:less`: Rule, Subject, Message).
- `exemption.go:CheckSorted` - rejects a file that is not in canonical order.
- `exemption.go:Set`/`NewSet`/`Consume`/`Unused` - multiset matching of findings to exemptions; `Unused` reports stale entries.
- `exemption.go:Diff` - splits current findings into missing and stale.
- Subdirectories: none.

## Invariants

- The identity triple is exact on purpose (`exemption.go` package comment).
- Saved files are always sorted; validation fails on an unsorted file with instructions to run `apitool validate --update` (enforced by the caller via `CheckSorted`).

## Side Effects

- Reads and writes the exemptions JSON file (`exemption.go:Load`, `exemption.go:Save`, mode 0644).

## Verification

- Run: `cd tools/apitool && go test ./...`
