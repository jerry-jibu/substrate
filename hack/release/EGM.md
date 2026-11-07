# release

Release drafting helpers. `draft-release.sh` creates a DRAFT GitHub release
for an existing tag; a maintainer edits and publishes it (see
`docs/dev/releasing.md`).

## Interface

- `draft-release.sh` - `hack/release/draft-release.sh [--dry-run] <tag>`. Generates notes from `.github/release.yml` via the generate-notes API, appends upstream-only Images/Install/kubectl-ate sections and the committer list from `hack/util/get-contributors.sh`, then creates the draft.

## Invariants

- The tag must match `vMAJOR.MINOR.PATCH[-PRERELEASE]`; a prerelease tag marks the release as prerelease (`draft-release.sh` SEMVER / SEMVER_PRE checks).
- A non-dry run requires the tag to exist; a dry run of a missing tag runs the notes up to `remotes/origin/main`, which must already be pushed to the repository (`draft-release.sh` tag checks).
- If a release for the tag already exists, the script prints the notes and leaves the release alone, so rerunning never overwrites edited notes (`draft-release.sh` header comment).
- Notes start at the newest final release below the tag: every rc of a version shares one starting point, and a final release diffs against the previous final release (`draft-release.sh` previous-tag logic).
- The Images/Install sections are appended only for the upstream repository, because only its release workflow publishes images (`draft-release.sh` REPO check).
- `exists` treats only HTTP 404/422 as "not found"; any other gh failure exits, so a rate limit is not mistaken for a missing tag (`draft-release.sh:exists`).

## Side Effects

- Calls `gh api` (tags, generate-notes) and `gh release create`; needs git, jq, and an authenticated gh with write access even for `--dry-run` (`draft-release.sh` header comment). Runs `hack/util/get-contributors.sh` against the local clone.

## Verification

- Run: `hack/verify/shellcheck.sh` (shellchecks every repo `.sh` script).
