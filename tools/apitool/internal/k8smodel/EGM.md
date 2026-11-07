# k8smodel

## Overview

Models Substrate's Kubernetes API from its generated CustomResourceDefinitions
(the YAML in `manifests/ate-install/generated`).

## Interface

- `k8smodel.go:Build(sources []string)` - parses one CRD manifest per source into a `k8smodel.go:API` with resources sorted by Kind; errors on zero sources or a CRD serving != 1 versions (`k8smodel.go:newResource`).
- `k8smodel.go:Resource` - Group, Version, Kind, Scope, schema `Doc`, and `Fields` (apiVersion/kind/metadata excluded).
- `k8smodel.go:Field` - serialized name, render-ready Type (`string`, `[]object`, `map[string]string`), doc (including item/value descriptions), Required, Default, Enum, and nested Fields.
- Subdirectories: none.

## Invariants

- Each CRD must serve exactly one version, the one modeled (`k8smodel.go:newResource`).

## Side Effects

- ["None"] (parses provided manifest text)

## Verification

- Run: `cd tools/apitool && go test ./...`
