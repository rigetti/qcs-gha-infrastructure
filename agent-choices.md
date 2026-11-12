# Agent choices

Assumptions and decisions made while adding release-plz support.
Decided by the user, not relitigated: replace knope with release-plz, and use `GITHUB_TOKEN` only (no PAT, no GitHub App).

## Versioning and compatibility

- Consumers pin tags such as `@v0.4.10`, and removing or changing inputs breaks them.
  So the knope workflows are **kept and marked deprecated** (header comment, README headings, migration section).
  Their removal is a follow-up breaking release.
- New workflows are additive: `release-plz.yml`, `release-plz-prerelease.yml`.
  `rust-cli-build.yml` gains an optional `version` input (default empty), which is not breaking.
- The macOS runner fix (`macos-14` to `macos-latest`) changes the default build matrix of an existing workflow.
  It is a `fix:` because `macos-14` cannot run at all; callers who pass their own `matrix` are unaffected.
  I also changed this repository's own `setup-rust` CI job, which used the same label.
- This repository's own release (`release.yml` calling `prepare-release.yml`, driven by `knope.toml`) is **unchanged**.
  Moving it needs a decision about how internal `@vX.Y.Z` pins get rewritten (release-plz has no `versioned_files`).
  The new workflows are registered in `knope.toml` so their internal pins keep being rewritten, and the CI lint check stays satisfied.

## Design of `release-plz.yml`

- One pipeline per push: `release` (tag and draft release), `build`, `publish`, then `release-pr`.
  No tag or release triggered workflow is used, because `GITHUB_TOKEN` events start no runs.
- Draft invariant preserved: a release is public only after its binaries are attached.
  release-plz creates the draft (`git_release_draft = true` in the caller's `release-plz.toml`), `gh release upload` attaches, `gh release edit --draft=false` publishes.
  The workflow checks the config before releasing and fails otherwise, and checks the release is still a draft and the asset count before publishing.
  Option names (`git_release_draft`, `git_release_type`, `git_only`, `release_always`) are from the current docs; the config parser rejects unknown keys, and I confirmed locally that these parse.
- `build` checks out `refs/tags/<tag>`, so binaries are built from exactly the tagged commit.
- `release-pr` needs `release` but not `build`, so the next release PR is not delayed by the build.
  Running it after `release` avoids opening a second PR for a version whose tag does not exist yet.
- `concurrency` with `cancel-in-progress: false`, per branch.
  A queued run can be replaced by a newer one; that is safe because `release` acts on the version in the tree.
- Default branch only: the workflow refuses other refs, since GitHub offers every branch for `workflow_dispatch`.
- Token fallback (`secrets.token || github.token`) is applied inside the called workflow.
  Job-level `permissions` declare the minimum; callers must grant at least `contents: write` and `pull-requests: write`.
- `git_only = true` is recommended for binaries not published on crates.io.
  Version state then comes from tags, which matches existing knope tags (`v0.1.x`).
- Only one package per repository ships binaries (`package`, default `bin`).
  Releases of other packages in a workspace are not undrafted by this workflow, so they must not be drafts.
- `release-plz` action pinned to the commit of `v0.5.148` (peeled from the annotated tag).
  Its default binary version is `0.4.3`, mirrored by the `release-plz-version` input.
  The action itself pins `cargo-semver-checks` and `cargo-binstall`.
- Recovery after a partial failure: re-run failed jobs of the same run.
  A new run finds the tag and releases nothing, leaving a draft.
  The sibling `todo-curator` PR adds a `workflow_dispatch` `tag` input to resume; not done here.
  Open question whether to align.

## Design of `release-plz-prerelease.yml`

- release-plz has no prerelease mode.
  Its release PR computes the next stable version, and `release` releases the version in the tree.
- Minimal, `GITHUB_TOKEN`-only flow: validate an explicit SemVer prerelease `version`, build with `release-plz set-version` applied in the runner only, create a draft prerelease, attach the binaries, publish.
- `set-version` fails without a `CHANGELOG.md`, so `rust-cli-build.yml` passes a temporary config with `changelog_update = false`.
  Verified locally in a scratch crate: exit 0, `Cargo.toml` updated, no changelog created.
- Nothing is pushed.
  The tag is created at the dispatched branch's head commit, which still has the old version in `Cargo.toml`.
  This cannot leak a prerelease version into the default branch, and needs no push permission on the branch.
- The version is explicit.
  Computing the next `rc.N` (as the sibling PR does with `release-plz update`) is a possible follow-up.
- Open: `git_only` version detection may see `v1.3.0-rc.0` tags.
  A tag on a branch that is not the default branch should not affect `release-plz` on the default branch, but I did not test this.

## Deprecated knope workflows

- `prepare-prerelease.yml` had `token` as `required: true`, no `|| github.token` fallback, and a top-level `contents: read` that capped the token.
  Fixed (token optional with fallback, `contents: write`), because it is not breaking and makes the deprecated flow usable without a PAT.
  `rust-cli-prerelease.yml` makes `token` optional to match.
  Untested: this needs a real run on a branch that accepts pushes from `github-actions`.
- `prepare-release.yml` still requires a real token on `push`, because knope pushes a commit to the default branch and `GITHUB_TOKEN` cannot trigger the build.
  Left as is: it is deprecated.

## Validation

- `actionlint` is not installed and downloading it was blocked, so it did not run.
- All workflow files parse with `yq`.
- Every `run:` block of the new and changed workflows passes `shellcheck` with `${{ }}` replaced by a placeholder.
- The draft-config check script was run against fixtures: draft true, missing, overridden by a package entry, set only on the package, and no config file.
- `release-plz set-version` and `update`, and config parsing (including rejection of unknown keys), were run locally with release-plz 0.4.3.
- Not run: any workflow in GitHub Actions.
  No release, draft release, tag, or prerelease was created, so `release-plz release` with a draft, the tag existing before `build`, `gh release upload/edit`, and `gh release create --prerelease` are untested.
  The repository has no fixture repository that consumes these workflows.

## Manual steps for humans

- In each calling repository, turn on "Allow GitHub Actions to create and approve pull requests".
- Approve the held `pull_request` runs of release PRs (`action_required`), or supply the optional `token` secret.
- Check how the first run behaves on a throwaway repository before moving a real one.
- After this merges and releases, pin callers to the new tag.

## Open design questions

- Align the prerelease design with `todo-curator` (computed `rc.N` versus explicit version)?
- Add the `tag` dispatch input for resuming a failed publish?
- Use `--latest` on `gh release edit` for stable releases (the sibling PR does)?
  GitHub's default `make_latest` for an update is expected to do the right thing, but this was not verified.
- Move this repository's own release to release-plz, and how to handle its internal pins?
- Schedule the removal of the knope workflows.
