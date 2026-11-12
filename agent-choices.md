# Agent choices: remove self-pins

Delete this file before merging.

Assumptions and choices made by Claude Code while preparing this change,
recorded for review.

## Basis

- Nested reusable workflows use `./.github/workflows/<file>.yml`.
  GitHub resolves that to the same commit as the calling workflow,
  also for a caller in another repository.
  Verified in <https://github.com/BatmanAoD/gha-experiments-consumer/actions/runs/43278803558>:
  a consumer calling `outer.yml@v1` ran the sibling from tag `v1`, not from `main`.
- A step-level `uses: ./actions/<name>` resolves against the caller's checkout
  (same run), so the composite actions cannot be referenced locally.
  They were inlined instead.

## Inlining `actions/setup-rust`

- The steps read their inputs from job-level `env`
  (`SETUP_RUST_TOOLCHAIN`, `SETUP_RUST_COMPONENTS`, `SETUP_RUST_TARGETS`,
  `SETUP_RUST_CACHE`, `SETUP_RUST_CACHE_KEY`)
  instead of `${{ inputs.* }}`.
  This keeps every copy byte-identical,
  so a check can compare them.
  Side effect: the variables are visible to the job's other steps,
  including a caller's `test-command`. Considered harmless.
- `${{ inputs.toolchain }}` interpolated into the shell script became
  `"${SETUP_RUST_TOOLCHAIN}"`. Same behavior; no interpolation into shell.
- The component and target loops still rely on word splitting,
  now with `# shellcheck disable=SC2086`, since actionlint shellchecks
  workflow `run:` blocks but never saw the composite action's.
  The three `>> "${GITHUB_ENV}"` lines were grouped for SC2129 for the same reason.
- The sccache steps were dropped, not inlined:
  no reusable workflow exposed an `sccache` input,
  so they were dead code in every caller here.
  The README keeps a one-paragraph summary of the measurement that made
  sccache opt-in.
- The cache path's `${{ inputs.sccache != 'true' && 'target' || '' }}`
  became a plain `target` for the same reason.
- The cache step is now `if: env.SETUP_RUST_CACHE == 'true'`.
  Jobs that never cached (`fmt`, `rust-cli-build.yml`) keep the step,
  skipped, so all copies stay identical.
- Steps are named `... (setup-rust)`; `scripts/check-inlined-steps`
  keys on that suffix.

## Inlining `actions/reset-changelogs`

- Inlined as one `run:` step named "Reset changelogs",
  identical in `prepare-release.yml` and `prepare-prerelease.yml`.
  `KNOPE_CONFIG` is fixed to `knope.toml` in the step's `env`:
  no workflow ever passed `knope-config`.
- `ci.yml` now extracts that step's script from `prepare-release.yml` with `yq`
  and runs it through the three original test cases,
  so it tests the shipped script rather than another copy.

## Keeping copies consistent

- New `scripts/check-inlined-steps` (run by the `lint` job; shellchecked with
  the other scripts) fails when:
  - any setup-rust copy differs from the first one found (`ci.yml`),
  - the first copy is not exactly the three expected steps,
  - a job carrying the steps does not set all five `SETUP_RUST_*` variables,
  - the two "Reset changelogs" steps differ.
- It supports both mikefarah's `yq` (on GitHub runners)
  and the jq-wrapper `yq` (local, Homebrew `python-yq`).
  Verified locally with the latter, including a negative test
  (mutated copies fail with three errors).
  The mikefarah path is verified by this PR's CI only.

## Self-pin guard and knope

- Removed the `lint` step that required every self-pinning file to be in
  `knope.toml`, and the `fetch-depth: 0` that only it needed.
- Added a `lint` step that fails on
  `uses: rigetti/qcs-gha-infrastructure/` (case-insensitive, optional quote)
  anywhere under `.github/` and, if it ever reappears, `actions/`.
  Before the change it found all 12 self-pins.
- Added a `lint` step that fails on a step-level `uses: ./...` under `.github/`
  other than `./.github/workflows/`,
  since a local action would pass this repository's CI and break callers.
- The guards are `grep`-based and miss exotic YAML
  (quoted `"uses"` key, folded scalar value);
  judged not worth a YAML-aware rewrite.

## Adversarial review

Two review agents (non-test and test code) found no behavior differences for
consumers. Fixed after review:
`scripts/check-inlined-steps` swallowed yq/jq errors,
ignored `.yaml` files, hard-coded the `SETUP_RUST_*` list
(now derived from the steps), and accepted a non-mapping job `env`;
stale sccache comment in `ci.yml`; README wording on permissions.
Left as is: the check keys on the `(setup-rust)` name suffix,
so renaming all three steps of a job, or inserting an unrelated step between
them, escapes it; and only the `run:` body of "Reset changelogs" is executed
in CI, not its `env`.
The experiment run link is cited in this PR, not in the README,
because Actions run logs expire.
- `knope.toml` keeps only the README badge in `versioned_files`.
  knope cannot run in the jj workspace itself
  (libgit2: `unsupported extension name extensions.relativeworktrees`),
  so it was run in a scratch Git repository holding `main` (tagged `v0.4.10`)
  plus this commit: `knope release --dry-run` (0.23.0) proposes `v0.5.0`
  and touches only `README.md` and `CHANGELOG.md`.
- The PR's `Release` dry-run job fails at checkout
  (`could not read Username for 'https://github.com'`).
  It fails the same way on `main` and on every pull request since
  2026-11-12 16:30 UTC, so it is unrelated to this change;
  most likely the `RELEASE_TOKEN` secret is expired or revoked.
  Not fixed here (no token or settings changes).
- `dependabot.yml` lists `/` only.

## Release and compatibility

- Commit type `refactor!` with a `BREAKING CHANGE:` footer:
  removing the documented composite actions is breaking,
  so knope should bump `0.4.10` to `0.5.0`.
- A `gh search code` across the `rigetti` org (done before this change)
  found no external callers of the composite actions.
- Tags `v0.4.10` and earlier keep their self-pins and their `actions/`,
  so consumers pinned to them are unaffected.
- The release carrying this change is cut by the existing `release.yml` flow;
  it needs no special handling, as there are no internal pins left to rewrite.
- This repository's own CI already called `prepare-release.yml` by `./` path,
  and `ci.yml` used `./actions/...`; nothing in CI referenced a tag.

## Not done

- `macos-14` to `macos-latest`: already on `main` (#23 merged as `8a378d90`),
  so there was nothing left to change.
- actionlint was not run locally (not installed); the `lint` job runs it.
