# lint

## Overview

API-shape rules ("lint") run against a `model.API`, catching violations of
Substrate's API style guidelines. Each rule returns one `Finding` per
violation.

## Interface

- `lint.go:Rule` - name, description, and `Check func(*model.API) ([]Finding, error)`; `lint.go:Finding` is `{Subject, Message}`.
- `lint.go:All` - every rule `apitool validate` runs, in report order: `resources.go:ResourceMetadata`, `standard_methods.go:ListMethodShape`, `naming.go:RequestNameMatchesMethod`, `standard_methods.go:StandardMethodReturnsResource`, `standard_methods.go:GetRequestShape`, `standard_methods.go:DeleteRequestShape`, `standard_methods.go:DeleteOptionsShape`, `standard_methods.go:CreateRequestShape`, `standard_methods.go:UpdateRequestShape`, `standard_methods.go:ListResponseNameMatchesMethod`, `naming.go:CustomResponseNameMatchesMethod`, `standard_methods.go:SingletonNoList`, `naming.go:SubresourceNaming`, `naming.go:EnumZeroValueUnspecified`, `naming.go:EnumValuesPrefixed`, `resources.go:ResourceStatusFieldShape`, `no_oneofs.go:NoOneofs`, `documented.go:Documented`, `doc_format.go:DocFormat`.
- `method.go:forEachStandardMethod`/`forEachResource`/`forEachMessage`/`forEachField`/`forEachMethod`/`forEachEnum`/`allOf` - traversal combinators building rule checks; `method.go:wantStandardName` derives standard-method names.
- `fields.go:checkFields` - declarative request-field shape checks (`fields.go:required`/`optional` with `typeMatcher`s).
- Subdirectories: none.

## Invariants

- A rule error (not findings) means the API is malformed in a way no rule can check past, such as a standard method that does not parse (`lint.go:Rule` doc).
- Rule names are stable identifiers: they key the exemptions file (`validate.AsExemptions`).

## Side Effects

- ["None"] (pure predicates over the in-memory model)

## Verification

- Run: `cd tools/apitool && go test ./...`
