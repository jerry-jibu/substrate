# docsite

## Overview

Renders the API reference pages for the project website: one HTML fragment
per version plus nav JSON, from the gRPC model, the Kubernetes model, and
the YAML examples.

## Interface

- `docsite.go:Render(api, examples, k8sAPI, k8sExamples)` - produces `docsite.go:Page` (`Body` HTML fragment + `Nav` JSON tree of `docsite.go:treeNode`s grouped as "gRPC API" and "Kubernetes API"); `api` must be scoped to the documented service.
- `docsite.go:Assets` - embedded `style.css` and `search.js` the fragment needs; `docsite.go:pageTemplate` is the embedded `template.html.tmpl`.
- `docsite.go:resourceView`/`methodView`/`attrNode` (with `docsite.go:maxAttrTreeDepth` 6) - per-page resolved views; `docsite.go:indexEntry` is the client-side search index row, JSON-escaped with encoding/json's default HTML escaping so a `</script>` in a doc comment cannot break out.
- `pages.go` - resource page construction (`TestResourcePages` covers it); `markdown.go` - doc-comment to HTML rendering; `highlight.go` - proto/YAML source highlighting (`TestCodeLines`, `TestHighlightYAML`); `k8s.go` - Kubernetes view building.
- Subdirectories: none.

## Invariants

- The search index keeps encoding/json's default HTML escaping on purpose (`docsite.go:indexEntry` comment).
- Render requires a service-scoped API (`docsite.go:Render` doc).
- Search index and nav anchors are element ids inside Body (`docsite.go:Page.Nav` doc).

## Side Effects

- ["None"] (in-memory rendering; the caller writes the files)

## Verification

- Run: `cd tools/apitool && go test ./...`
