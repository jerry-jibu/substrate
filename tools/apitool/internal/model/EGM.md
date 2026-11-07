# model

## Overview

Compiles a proto3 source file with protocompile and exposes the data model
of the Substrate API: services, methods (with their
`substrate.ateapi.v1.method` resource annotations), messages, fields, and
enums, flattened and queryable.

## Interface

- `model.go:Build(ctx, source)` - compiles one proto file into a `model.go:API`.
- `model.go:API` - `Services`, `Messages`, `Enums` (nested types flattened); `model.go:API.MessagesByFullName` indexes messages.
- `model.go:Service` - `FullName`, `Name`, doc `Comment`, declaration-ordered `Methods`.
- `model.go:Method` - name, comment, `InputName`/`OutputName`, `Resource` (from the method annotation), declaring service.
- `model.go:Message`/`model.go:Enum`/`model.go:Field` - full names, short names (nested dotted), comments, fields.
- `model.go:ScopeToService` - narrows an API to one service (used by both `validate` and `docsite`).
- `model.go:GlobalParent` - the resource-annotation parent standing for the whole installation.
- Subdirectories: none.

## Invariants

- `Build` compiles exactly one file; its logical name is an internal label only (`model.go:sourceFileName` doc).

## Side Effects

- ["None"] (compilation happens in memory; reads the provided source)

## Verification

- Run: `cd tools/apitool && go test ./...`
