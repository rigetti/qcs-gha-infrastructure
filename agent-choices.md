# Agent choices

Delete this file before merging.

Assumptions and choices made by Claude Code while preparing two stacked PRs,
recorded for review:

- Part 1: #25, remove self-pins (the base).
- Part 2: #24, release-please prototype (stacked on #25).

# Part 1: remove self-pins (#25)

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

# Part 2: release-please prototype (#24)

In the order the decisions came up.

## Rebase onto #25

#24 was first written against `main` and later rebased onto #25.
Changes forced by the rebase:

- Dropped every `x-release-please-start-version`/`x-release-please-end` block
  around workflow self-pins: #25 removed the pins themselves.
  Each conflicting workflow file was taken from #25 unchanged.
- The prototype config's `extra-files` is now the README badge only.
  The `.github/workflows/*.yml` glob is gone,
  so no extra-file lies under `.github/workflows/`,
  and this repository no longer needs a token with the `workflows` permission.
  The general caveat for consumers whose version files are workflow files stays
  (README, `docs/release-please.md`, `release-please-prerelease.yml`).
- Dropped the composite action `actions/release-please-lint`:
  #25's guard fails on a step-level local action,
  and the README states that this repository ships no composite actions.
  `scripts/lint-release-pins` stays as the tool, for callers and for the tests.
- Dropped the CI step that linted this repository's pins.
  With the pins gone, its pin scan has nothing to find
  (the self-pin guard already forbids them),
  and without a manifest the version comparison is trivial on a single line.
  The only remaining value, catching a badge that loses its annotation,
  is covered by `scripts/test-release-please-bump`:
  it lints the bumped copy with `--no-scan` against the new version,
  which fails when no annotated version exists.
  Considered and rejected: a direct `scripts/lint-release-pins --no-scan` step
  (redundant with the above), and a comparison of the badge against the
  prototype manifest (fails after the next knope release, which moves the badge
  but not the manifest).
- `scripts/test-release-please-bump` now checks the badge, not a workflow pin,
  for the kept `v` prefix.
- No new workflow calls another workflow of this repository,
  so the `./.github/workflows/` convention did not need applying.

## Scope

- Prototype beside the knope workflows.
  No knope workflow, `knope.toml`, or behavior of an existing workflow changed.
- The only edits to existing files are the annotation on the README badge
  (so the prototype config has something real to rewrite), a README section,
  and CI steps.
  The badge's marker sits after the `)`, outside knope's regex.
- The prototype config lives at `.github/release-please/config.json` and `manifest.json`, not at the repository root, so it is not mistaken for the config of a live release flow.
  The CLI takes `--config-file` and `--manifest-file`, and the workflows expose both as inputs.

## CLI, not `release-please-action`

- The workflows run `npx release-please@17.11.2` (latest on the registry when written; the version is an input).
- Reasons: separate `release-pr` and `github-release` commands (the action runs both per invocation unless told to skip one), a `--draft` flag on `github-release`, a one-shot `--release-as` flag, and no third-party action to SHA-pin.
- Cost: no lockfile for release-please's transitive dependencies; `npx` resolves them at run time.
  A consumer that wants them frozen can vendor an `npm ci` step.
  Worth revisiting.
- `googleapis/release-please-action` is at v5.0.0 (checked with `gh api`); not used.

## Dispatch-only, draft first

- Every entry point is `workflow_call`, meant for a caller's `workflow_dispatch`.
  Nothing triggers on push.
- release-please's draft mode creates the release with no tag; GitHub creates the tag, at `target_commitish`, when the draft is published.
  So builds use the draft's commit SHA (an output), not a tag.
  This is better than the release-plz design in one respect: no tag exists until the assets are attached.
- Publishing is a `PATCH` with `draft=false`.
  No `make_latest` is sent.
  GitHub's default then decides "Latest", which can mark an older version as latest when backporting.
  Open question for the user.
- `release-please` is never run with `--prerelease`: it marks every 0.x release a prerelease (`prerelease && (preRelease || major === 0)` in `manifest.js`).
  The publish workflow sets `prerelease` itself when the tag has a suffix.

## Idempotency

- The draft workflow derives `v<manifest version>` and looks for an existing release (list endpoint; `releases/tags/` hides drafts) before running release-please, because release-please only acts on a merged PR it has not yet labeled `autorelease: tagged`.
- States: published (skip), draft (reuse), none (run release-please).
  After that, a tag with no release gets a draft created directly; no tag means nothing is pending.
- Publish deletes and re-uploads same-named assets, verifies the draft holds every file, then publishes.
  Already published is a no-op.
- Prerelease: the `rc` input continues a failed run; the tag is pushed only if absent; the draft is created only if missing.

## Hooks for builds

- A reusable workflow cannot accept another workflow as an input, so a "build-workflow" hook is not possible.
  Hook = any job in the caller between `draft` and `publish` that checks out `needs.draft.outputs.sha` and uploads `release-*` artifacts.
- `release-assets-build.yml` is a generic matrix-of-commands convenience.
  It requires a full commit SHA as `ref`.
- `rust-cli-build.yml@v0.4.10` is not called from any new workflow (lesson from release-plz: its `ref` input is released, but its default matrix still has `macos-14`, which GitHub rejects).
  Documented as something callers can use with an explicit `matrix`.
- New workflows reference no other workflow of this repository, so there is no bootstrap order.
  Callers use `@prototype-release-please` until a tag exists.

## The lint

- The lint mirrors the `Generic` updater (inline, block, first semver) and compares against the manifest when given one.
- Fixtures cover the pin scan and the manifest comparison;
  these remain meaningful for callers whose own files pin a repository by tag.
- The README is an extra-file, so it must not quote a start marker (it would open a block to the end of the file).
  Marker documentation is in `docs/release-please.md`, which is not an extra-file.
  The lint reports a marker with no semver on its line, which catches prose.
- The README's illustrative `@v0.3.0`-style example pins are not annotated and not extra-file versions.

## Tokens

- release-please writes the release PR branch through the API.
  GITHUB_TOKEN cannot create or update files under `.github/workflows/`.
  Confidence about 85%; from GitHub's documentation of the permission; not tried (no tokens may be created here).
- Before the rebase onto #25, this repository's config listed its workflow files,
  so it needed a token with the `workflows` permission.
  It no longer does: the only extra-file is `README.md`.
- The `token` secret exists in every workflow for consumers that do list workflow files,
  with the fallback `secrets.token || github.token` inside the called workflow.
  The same limit applies to the prerelease flow's tag push.

## Prerelease

- Experimental, because it cannot be run here.
- Reuses release-please's own updater classes (internal module paths under `build/src`) so version files match what a release PR would write.
  Pinned to the release-please version, which is why the test runs the step verbatim.
  These internals are not a public API and could move.
- Tag-only: the bump commit is local and never reaches a branch.
- Alternative considered: `release-pr --release-as X.Y.Z-rc.N` and tag the PR head.
  Rejected: release-please would then compute later versions in an unpredictable way from a prerelease tag, and it was not possible to try.

## Verified and not verified

Verified here (after the rebase):

- `yq` parses every workflow; `shellcheck` is clean for the scripts and for every `run:` block (extracted, with `${{ }}` replaced).
- `lint-release-pins`: the fixtures, passing and failing, also under bash 3.2 (`/bin/bash`).
- release-please 17.11.2's `Generic` updater over this repository's real files: the badge rewritten, `v` kept, no other line changed; the prerelease workflow's bump step run verbatim.
- #25's self-pin and local-action guards, and `scripts/check-inlined-steps`, pass on the stacked branch.
- `release-please github-release --help` and `release-pr --help` confirm the flags used (before the rebase; unchanged since).

Not verified:

- Any release-please workflow run on GitHub (no dispatch against the real repository was allowed).
  `actionlint` is not installed here; CI installs it and will check the workflows.
- `release-please release-pr` and `github-release` end to end, `--draft` behavior, the label flow, and the `issues: write` need.
- The GITHUB_TOKEN workflow-file limitation, and the prerelease flow.
- Whether `fs.globSync` (experimental in Node 22) is stable enough; Node 24 is the alternative if CI complains.

## Incident

During the experiments, `release-please release-pr --local --local-path` was pointed at this workspace.
It ran `git checkout main` and `git reset --hard origin/main` there and discarded the uncommitted work.
I restored it from the jj snapshot (`jj restore --from`); no other workspace was touched.
Lesson recorded: never use `--local-path` on a workspace.
