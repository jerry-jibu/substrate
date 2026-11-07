# site-content

## Overview

Root-module Go tool that generates Hugo content stubs in `site/content/`
from the Markdown under `docs/` and `demos/*/README.md`, so `site/content/`
does not need to be checked into the repository.

## Interface

- `main.go:main` - flags `--repo` (repository root, default `.`) and `--out` (Hugo content directory, required); exits 2 without `--out`.
- `main.go:generate` - removes and regenerates the output directory, writes section index files from `main.go:sections` (slugs get-started, concepts, tutorials, guides, reference, security, project, each with Title/Weight/Description and optional Aliases), then walks `docs/` mapping each Markdown file into its section per front matter; `main.go:frontMatterRE` and `main.go:h1RE` parse front matter and the H1 title.
- Subdirectories: none.

## Invariants

- Generated content is disposable: the tool wipes `--out` before regenerating (`main.go:generate` `os.RemoveAll`).
- A doc's section comes from its front matter; the section slugs are fixed in `main.go:sections`.

## Side Effects

- Deletes and rewrites the `--out` directory tree (`main.go:generate`).

## Verification

- Run: `go test ./tools/site-content/...` (root module; this tool has no go.mod of its own)
