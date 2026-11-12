# release-please guide (prototype)

Details behind the README section on release-please.
Everything here is a prototype: it sits beside the knope workflows and does not replace them.

## Annotating files

release-please's `generic` updater rewrites a semver only on a line it was told about.
Two forms, both written as a comment in the file's own syntax.

Inline, on the line itself:

```yaml
uses: my-org/my-repo/actions/setup@v1.2.3 # x-release-please-version
```

```markdown
[![release](...)](https://github.com/my-org/my-repo/releases/tag/v1.2.3) <!-- x-release-please-version -->
```

Block, around lines that cannot carry a trailing comment:

```yaml
# x-release-please-start-version
version: 1.2.3
appVersion: 1.2.3
# x-release-please-end
```

Rules, checked against release-please 17.11.2's `Generic` updater:

- The scope after `x-release-please-` is one of `version`, `major`, `minor`, `patch`, `date`, `version-date`.
  `version` replaces a whole semver; `major`, `minor`, `patch` replace the first bare integer on the line.
- **The first semver on the line wins.**
  A line like `run: tool 9.9.9 --with my-org/x@v1.2.3` has its `9.9.9` rewritten and the pin left alone.
  Keep annotated lines to one version.
- The `v` in `@v1.2.3` is not part of the match and survives.
  A prerelease or build suffix (`1.2.3-rc.1`, `1.2.3+build`) is part of it and is replaced too.
- Inside a block, every line has its first semver rewritten.
  Wrap only the lines you want changed: a block around a third-party `uses: foo/bar@sha # v7.0.1` rewrites that comment.
- A marker quoted in prose is a marker.
  A README sentence that names the inline marker next to a version number gets that number rewritten.
  A start marker with no end marker rewrites the file from there to the end.
  That is why this guide is not one of the config's extra-files.
- A line with no marker is skipped silently.
  `scripts/lint-release-pins` (below) is the guard.

### Other updaters

An extra-file entry is a bare path or an object with a `type`.

| `type` | Addresses | Needs | Notes |
|---|---|---|---|
| `generic` | annotated lines | | The annotation forms above. |
| `json` | JSONPath | `jsonpath` | e.g. `$.version`, `$.packages[0].version`. |
| `yaml` | JSONPath | `jsonpath` | Comments survive; quoting around the value does not. |
| `toml` | JSONPath | `jsonpath` | e.g. `$.package.version`. |
| `xml` | XPath | `xpath` | e.g. `//project/version`. |
| `pom` | Maven `pom.xml` | | Maven-specific; see the release-please docs. |

Add `"glob": true` to an entry to let `path` be a glob (`.github/workflows/*.yml`).

A bare string entry ending in `.json`, `.yaml`, `.yml`, `.toml`, or `.xml` runs the path-based updater on `$.version` (`/*/version` for XML) **and** the generic one.
Any other bare string runs only the generic one.
A path that matches nothing in a file is a logged no-op, not an error.

## Config and manifest

`release-please-config.json` sets the behavior, `.release-please-manifest.json` records the current version per package.
The manifest is what the draft workflow reads to find the version, so merging the release PR (which bumps it) is what makes a release pending.

The `release-type` picks a strategy: which language files it updates by default and which changelog it writes.
`release-please` 17.11.2 accepts:
`simple`, `node`, `python`, `rust`, `go`, `helm`, `maven`, `java`, `dart`, `dotnet-yoshi`, `elixir`, `expo`, `krm-blueprint`, `ocaml`, `php`, `r`, `ruby`, `salesforce`, `sfdx`, `terraform-module`, `bazel`, and Google-internal variants (`*-yoshi`, `*-librarian`, `java-bom`, `java-lts`, `java-backport`).
For a repository that is not one of those, or that needs several, use `simple` and list every versioned file as an extra-file.
`simple` writes `CHANGELOG.md` and a `version.txt` it updates only if the file exists.

Keys that matter here (the schema lists more):

- `include-component-in-tag: false` and `include-v-in-tag: true`: tags are `vX.Y.Z`.
  The draft workflow derives the tag from the manifest, so it needs this.
- `bump-minor-pre-major`: while below 1.0.0, a breaking change bumps the minor.
- `bump-patch-for-minor-pre-major`: while below 1.0.0, a feature bumps the patch.
- `changelog-sections`, `skip-changelog`, `changelog-path`.
- `prerelease`: **do not set this for a 0.x project.**
  release-please then flags every `0.x` release as a prerelease on GitHub.
  The draft workflow omits `--prerelease`, and the publish workflow marks a version with a suffix as a prerelease itself.
- `release-as`: sticks in the config until removed.
  The `release-as` input of `release-please-pr.yml` is one-shot instead.

## Conventional commits

| Commit | Bump |
|---|---|
| `fix:` and other types the changelog shows (by default also `perf:`, `revert:`, `deps:`) | patch |
| `feat:` | minor |
| `feat!:`, any type with `!`, or a `BREAKING CHANGE:` footer | major (minor below 1.0.0 with `bump-minor-pre-major`) |
| `docs:`, `chore:`, `ci:`, `test:`, `style:`, `refactor:`, `build:` | hidden from the changelog by default; expected to trigger no release alone (not verified against a live repository) |
| `Release-As: X.Y.Z` footer | forces that version |

Non-conventional commit messages are ignored.
The PR title of a squash-merge is the commit message, so lint it.

## Running a release

Caller workflow (a dispatch-only `release.yml` in the consumer):

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      step:
        type: choice
        options: [open-pr, release]
        description: open-pr first; release after merging the PR it opened.

permissions: {}

jobs:
  pr:
    if: inputs.step == 'open-pr'
    permissions:
      contents: write
      pull-requests: write
      issues: write
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/release-please-pr.yml@vX.Y.Z

  draft:
    if: inputs.step == 'release'
    permissions:
      contents: write
      pull-requests: write
      issues: write
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/release-please-draft.yml@vX.Y.Z

  build:
    needs: draft
    if: needs.draft.outputs.state == 'draft'
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/release-assets-build.yml@vX.Y.Z
    with:
      ref: ${{ needs.draft.outputs.sha }}
      version: ${{ needs.draft.outputs.version }}
      tag: ${{ needs.draft.outputs.tag }}
      matrix: >-
        [{"name": "linux", "runs-on": "ubuntu-latest", "run": "make dist", "path": "dist/*"},
         {"name": "macos", "runs-on": "macos-latest", "run": "make dist", "path": "dist/*"}]

  publish:
    needs: [draft, build]
    if: needs.draft.outputs.state == 'draft'
    permissions:
      contents: write
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/release-please-publish.yml@vX.Y.Z
    with:
      tag: ${{ needs.draft.outputs.tag }}
```

`vX.Y.Z` is the first tag of this repository that contains these workflows.
Until then, reference a branch (`@prototype-release-please`).

A Rust CLI project can replace `build` with `rust-cli-build.yml` (`bin`, `ref: ${{ needs.draft.outputs.sha }}`), which uploads `release-<target>` artifacts the publish workflow picks up.
Pin it to a tag that has the `ref` input and a `macos-latest` default; v0.4.10 still lists the unsupported `macos-14` in its default matrix, so pass a `matrix`.

A project with no assets drops `build` and passes `require-assets: false` to `publish`.

A custom build hook is any job that runs after `draft`, checks out `needs.draft.outputs.sha`, and uploads artifacts named `release-*`.
(A reusable workflow cannot take another workflow as an input, so the hook is a job in the caller.)

### Prerelease (experimental)

```yaml
  prerelease:
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/release-please-prerelease.yml@vX.Y.Z
    permissions:
      contents: write
    with:
      version: 1.3.0
  # build and publish as above, with
  #   ref: ${{ needs.prerelease.outputs.sha }}
  #   tag: ${{ needs.prerelease.outputs.tag }}
```

Dispatch it on the branch with the unmerged work.
It computes `1.3.0-rc.N` from the tags that exist, applies the config's extra-files in a local commit that is never pushed to a branch, pushes only the tag, and creates a draft prerelease.
A binary built from the tag therefore reports the version its tag carries.

Limits, none of which could be tried here:

- GITHUB_TOKEN cannot push a commit that touches `.github/workflows/**`.
  A config that lists workflow files as extra-files needs the `token` secret to carry the `workflows` permission.
- Releases made later by release-please are computed from the manifest, not from the rc tags, which sit on a commit no branch holds.
  This should be harmless; it is untested.

## Token caveats

- Default is the job's `GITHUB_TOKEN`; an optional `token` secret is preferred when supplied (`secrets.token || github.token`, resolved inside each called workflow).
- The repository setting **Allow GitHub Actions to create and approve pull requests** must be on, or opening the release PR fails.
- A PR created with `GITHUB_TOKEN` gets its `pull_request` runs held in `action_required`.
  Approve them.
- The checks go stale when release-please pushes again (after more commits land).
  Required checks then never report on the new head.
  Close and reopen the PR to start them.
- `GITHUB_TOKEN` cannot create or update files under `.github/workflows/`.
  release-please writes the release PR's branch through the API, so a config whose extra-files include a workflow file needs the `token` secret to hold the `workflows` permission.
  Confidence about this: about 85%, from how GitHub documents the permission, and untested here.
  This repository's own config does not: no workflow here pins this repository by tag, so only the README badge carries the version.
- Publishing a release with `GITHUB_TOKEN` does not trigger `release: published` workflows.
  Do the follow-up work in the same run.
- The called workflows take their permissions from the caller's `permissions:`; a caller that does not grant `contents`, `pull-requests`, and `issues` write fails.

## The lint

`scripts/lint-release-pins` fails when:

1. a line matching `--pattern`, under `--scan` paths, has no annotation;
2. a file holding such a line is not a generic extra-file in the config;
3. a version release-please would rewrite differs from the manifest (or, with no manifest, from the other pins);
4. a block is never closed, or a version marker has no semver on its line.

It is a script rather than an action, because this repository ships no composite actions.
A caller runs it from a checkout of this repository at a pinned tag, or copies it.
This repository's own CI does not run it directly:
its only annotated line is the README badge,
and `scripts/test-release-please-bump` already lints the bumped copy.

`scripts/test-lint-release-pins` runs it against passing and failing fixtures.
`scripts/test-release-please-bump` runs the prerelease workflow's bump step over this repository's real files and lints the result.

## Trying it on a throwaway repository

1. Create a private repository `<you>/rp-sandbox`.
   Settings, Actions, General: enable *Allow GitHub Actions to create and approve pull requests*.
2. Commit `release-please-config.json` (`release-type: simple`, a `packages` entry with `extra-files`) and `.release-please-manifest.json` (`{".": "0.0.0"}`).
   Add a file with an annotation, e.g. a `README.md` badge.
3. Add the caller workflow above, referencing `rigetti/qcs-gha-infrastructure/...@prototype-release-please`.
   (The workflows are public; no tag exists for them yet.)
4. Commit `feat: first` to the default branch, dispatch with `step: open-pr`.
   Expect a PR bumping the manifest, changelog, and annotated files to `0.1.0`.
   Approve its held runs or close and reopen it.
5. Merge it, dispatch with `step: release`.
   Expect a draft, then assets, then a published release `v0.1.0` whose tag points at the merge commit.
6. Dispatch `step: release` again: it reports the release as published and does nothing.
