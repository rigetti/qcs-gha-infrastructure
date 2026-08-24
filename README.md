# qcs-gha-infrastructure

Shared GitHub Actions and reusable workflows for QCS repositories.

## Composite actions

### `actions/setup-rust`

Installs a Rust toolchain with rustup and sets up compilation caching.

```yaml
- uses: rigetti/qcs-gha-infrastructure/actions/setup-rust@main
  with:
    toolchain: stable        # optional
    components: clippy       # optional, space-separated
    targets: x86_64-unknown-linux-musl  # optional, space-separated
    cache: "true"            # optional; "false" for untrusted code
    cache-key: clippy        # optional cache discriminator
    sccache: "false"         # optional; see the caching notes below
```

**Caching.** By default this caches the cargo registry and `./target` with
`actions/cache`.

Setting `sccache: "true"` instead installs
[sccache](https://github.com/mozilla/sccache) via
`mozilla-actions/sccache-action` and sets `RUSTC_WRAPPER=sccache` with
`SCCACHE_GHA_ENABLED=true`. To answer the obvious question: yes, sccache's
`gha` backend is **GitHub's own Actions cache service** under the hood — the
same backing store `actions/cache` writes to, reported as
`Cache location  ghac, …` in the post-run stats. There is no external cache
infrastructure to run. In principle its granularity beats a target-directory
archive, which one dependency bump invalidates wholesale. `CARGO_INCREMENTAL=0`
is set either way, which sccache requires.

In practice, measured on `todo-curator` (sccache 0.17.0, four platforms):

| | target-dir cache | sccache |
|---|---|---|
| `clippy` | 72s | 108s |
| `test` | 102s | 119s |
| integration jobs | 36–49s | 123–130s |

Hit rate was **0%** across repeat runs of identical code, with ~151 `Cache write
errors` per job against 194 successful writes. Two things contribute: sccache
cannot cache `cargo check` or `clippy` at all (they only emit metadata), and
whatever is failing those writes also prevents reuse in the full builds. So this
is off by default, and worth revisiting for build-dominated jobs only — check
the post-run stats block rather than assuming it helps.

Both paths respect `cache: "false"`, which disables caching entirely — use that
when building untrusted code, so its artifacts never land in a cache that a
later trusted run would read.

## Reusable workflows

### `rust-ci.yml`

`cargo fmt --check`, `cargo clippy -- --deny warnings`, and the test suite
(`cargo nextest run` by default) as three parallel jobs.

```yaml
jobs:
  rust:
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/rust-ci.yml@main
    with:
      test-command: cargo nextest run --all-features
```

### `rust-cli-build.yml`

Builds a release binary per platform and uploads each as an artifact named
`release-<target>` holding `<bin>-<target>`. Defaults to musl Linux on both
architectures, arm64 macOS, and x86_64 Windows.

```yaml
jobs:
  build:
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/rust-cli-build.yml@main
    with:
      bin: my-cli
      # Optional; override to add or drop platforms.
      matrix: |
        [
          {"runs-on": "ubuntu-latest", "target": "x86_64-unknown-linux-musl"},
          {"runs-on": "macos-14",      "target": "aarch64-apple-darwin"}
        ]
```

`musl-tools` is installed and `CC_<target>` set automatically for musl targets.
`CARGO_TARGET_DIR` is moved under `RUNNER_TEMP` so deep dependency trees do not
overrun Windows' path limit.

### `rust-cli-release.yml` and `rust-cli-prerelease.yml`

The whole **release-with-assets** pattern, for a repository that ships
binaries. Most callers want these two rather than the pieces they compose
(`rust-cli-build.yml`, `prepare-*.yml`, `publish-release.yml`).

```yaml
# .github/workflows/cli-release.yml — on merge to the release branch
on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  release:
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/rust-cli-release.yml@v0.3.0
    with:
      bin: my-cli
    secrets:
      token: ${{ secrets.RELEASE_TOKEN }}
```

```yaml
# .github/workflows/prepare-prerelease.yml — cut a release candidate
on:
  workflow_dispatch:
    inputs:
      prerelease-label:
        description: "Prerelease label; defaults to 'rc'."
        required: false
        default: ""

jobs:
  prerelease:
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/rust-cli-prerelease.yml@v0.3.0
    with:
      bin: my-cli
      prerelease-label: ${{ inputs.prerelease-label }}
    secrets:
      token: ${{ secrets.RELEASE_TOKEN }}
```

with a knope.toml split into the two halves:

```toml
[package]
versioned_files = ["Cargo.toml", "Cargo.lock"]
changelog = "CHANGELOG.md"
assets = "release/*"

[github]
owner = "rigetti"
repo = "my-cli"

[[workflows]]
name = "draft-release"
# PrepareRelease, then `git commit`, then `git push`. No Release step.

[[workflows]]
name = "publish-release"
# The Release step alone.
```

**Why it is built this way.** With `assets` configured, knope's `Release` step
creates the release as a draft, uploads the assets, and publishes only then. Run
at prepare time it would publish before any binary exists, and for the minutes a
cross-platform build takes, every consumer pinning the new tag gets
`could not find a release asset after filtering for valid extensions` from ubi —
or an empty release page. Deferring `Release` until the artifacts exist closes
that window for everyone, not just for CI.

`rust-cli-prerelease.yml` runs all three stages in one workflow because a
prerelease commit lands on a feature branch, where a push-triggered release
workflow watching the default branch can never fire. It also passes
`ref: ${{ github.ref }}` to every stage after the first: on
`workflow_dispatch`, checkout otherwise uses the commit the run started at,
which predates the bump.

### `rust-integration-tests.yml`



Runs a test command that needs credentials, optionally under a GitHub
environment so that untrusted code waits for a maintainer's approval.

Secrets — including environment secrets — are never passed to `pull_request`
runs from forks, so credentialed tests cannot be approved into a fork PR on
that trigger. Instead, call this workflow twice: once from `pull_request` /
`push` for code that is already trusted, and once from `pull_request_target`
for forks, passing an `environment` that has required reviewers.

```yaml
# .github/workflows/ci.yml — trusted code, runs immediately
jobs:
  integration:
    if: github.event.pull_request.head.repo.full_name == github.repository || github.event_name == 'push'
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/rust-integration-tests.yml@main
    with:
      test-command: cargo test --test api_integration_test --no-fail-fast
    secrets:
      github-api-token: ${{ secrets.MY_GH_TOKEN }}
```

```yaml
# .github/workflows/integration-fork.yml — forks, gated on approval
on:
  pull_request_target:
jobs:
  integration:
    if: github.event.pull_request.head.repo.full_name != github.repository
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/rust-integration-tests.yml@main
    with:
      ref: ${{ github.event.pull_request.head.sha }}
      environment: fork-integration-tests
      test-command: cargo test --test api_integration_test --no-fail-fast
    secrets:
      github-api-token: ${{ secrets.MY_GH_TOKEN }}
```

Create the environment (Settings → Environments) with **Required reviewers**
set. The job then waits for approval before checking out or running anything.

**Security:** the fork-facing caller runs untrusted code with your secrets in
scope once approved. The reviewer is the whole control. Read the diff — build
scripts, proc macros, `Cargo.toml` patches, and test fixtures included — before
approving, and pin the environment's reviewers to maintainers only.

### `prepare-release.yml`

The automatic half of a knope release. Nothing to configure, and no way to run
it by hand:

| Caller's event | Behavior |
|---|---|
| `pull_request` | `knope release --dry-run`, so the PR shows what it would release |
| `push` to the release branch | full release: bump, changelog, commit, push, publish |
| `push` of knope's own release commit | skipped |

That last row matters: the release push has to be made with a PAT so it can
trigger a binary build, and a PAT push also re-enters this workflow. The job is
skipped when the head commit starts with `release-commit-prefix` (default
`chore: prepare release`) — a successful release, not something to report as a
failure. Change the input if knope.toml uses a different commit message.

```yaml
name: Prepare release
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  prepare-release:
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/prepare-release.yml@main
    secrets:
      token: ${{ secrets.RELEASE_TOKEN }}
```

`token` must be a PAT or app token if the release push has to trigger anything
downstream — a binary build, say. Pushes made with `GITHUB_TOKEN` do not start
new workflow runs.

### `publish-release.yml`

The second half of a **split release**, for repositories that attach build
artifacts. The caller's `draft-release` knope workflow prepares and pushes the
version bump; this runs knope's `Release` step afterwards, once the artifacts
exist.

```yaml
jobs:
  build:
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/rust-cli-build.yml@v0.2.0
    with:
      bin: my-cli

  publish:
    needs: build
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/publish-release.yml@v0.2.1
```

Pass `ref: ${{ github.ref }}` to both when an earlier job in the same run
pushed the version bump — on `workflow_dispatch`, checkout otherwise defaults
to the commit the run started at, which predates that push, and knope would
find nothing to release.

with knope.toml split to match:

```toml
[package]
assets = "release/*"

[[workflows]]
name = "draft-release"
# PrepareRelease, commit, push — no Release step.

[[workflows]]
name = "publish-release"
# The Release step alone.
```

**Why split it.** With `assets` configured, knope's `Release` step creates the
release as a draft, uploads the assets, and only then publishes. Drafts are not
publicly visible, so the release never exists in the assetless state that makes
it uninstallable — ubi reports that as `could not find a release asset after
filtering for valid extensions`, and the window lasts as long as a
cross-platform build. Publishing at prepare time instead leaves every consumer
pinning the new tag broken for those minutes.

Point `prepare-release.yml` and `prepare-prerelease.yml` at the first half with
their `knope-workflow` input:

```yaml
    with:
      knope-workflow: draft-release
```

### `prepare-prerelease.yml`

The manual half: cuts a release candidate from the branch it is dispatched on,
so it can be installed and tested before merging. Wire it to
`workflow_dispatch` only.

```yaml
name: Prepare prerelease
on:
  workflow_dispatch:
    inputs:
      prerelease-label:
        description: "Prerelease label; defaults to 'rc'."
        required: false
        default: ""

jobs:
  prepare-prerelease:
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/prepare-prerelease.yml@main
    with:
      prerelease-label: ${{ inputs.prerelease-label }}
    secrets:
      token: ${{ secrets.RELEASE_TOKEN }}
```

The tag lands on the dispatched branch. Dispatching from the release branch
fails in the first step, before checkout: merging to it already releases, so a
prerelease there would duplicate or race that. GitHub offers every branch in the
Run workflow dropdown with no way to restrict the list, which is why this is a
workflow-level check rather than configuration. `default-branch` (default
`main`) names the branch to refuse.

### `actions/reset-changelogs`

Restores every changelog named in `knope.toml` to its state on the default
branch. Used by `prepare-release.yml` and `prepare-prerelease.yml` before knope
runs; rarely called directly.

```yaml
- uses: rigetti/qcs-gha-infrastructure/actions/reset-changelogs@v0.4.0
  with:
    default-branch: main        # optional
    knope-config: knope.toml    # optional
```

**Why.** Every prerelease runs `PrepareRelease`, and `PrepareRelease` writes the
pending changes into the changelog. Cut three release candidates from one branch
and the third changelog entry contains the first two as well — and that
accumulation lands on the default branch when the branch merges. Restoring the
changelog first means each run writes only the entry it would have written on
its own.

A changelog that does not exist on the default branch is deleted rather than
restored: it is new in this branch, so its correct prior state is absent.
Ported from the `knope.yaml` job template in the GitLab `qcs-infrastructure`
repository, which used `tomlq`; this reads `knope.toml` with the runner's own
Python instead, so nothing needs installing.

## Pinning

**Every `uses:` in this repository is either a commit SHA or a tag in a
repository we control. Keep it that way.**

### Why third-party actions are pinned by SHA

`actions/checkout@v4` is not a version. It is a *mutable pointer*: the
maintainers move the `v4` tag with each 4.x release, and Actions re-resolves it
when the job starts. Referencing it delegates, to someone outside this
organization, the decision about what code runs against our runners and our
secrets — at a moment we do not choose.

That is not hypothetical. In March 2025 an attacker retargeted the version tags
of `tj-actions/changed-files` at code that dumped runner memory, secrets
included, into public build logs — CVE-2025-30066, along with a related
compromise of `reviewdog/action-setup`. See CISA's alert:
<https://www.cisa.gov/news-events/alerts/2025/03/18/supply-chain-compromise-third-party-tj-actionschanged-files-cve-2025-30066-and-reviewdogaction>.
Repositories referencing the mutable tags picked it up on their next run, having
changed nothing. Those pinned by SHA were unaffected, because a commit SHA is
content-addressed and cannot be moved.

So references look like this, with the version in a trailing comment for
readability:

```yaml
uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4
```

The obvious cost is that security patches stop arriving on their own.
**Pinning without automated bumps is a downgrade, not an upgrade** — it trades a
supply-chain risk for silent staleness. Consumers should run Dependabot with the
`github-actions` ecosystem enabled; it parses exactly this form and rewrites the
SHA and the comment together.

This applies to `actions/*` as well as to everything else. GitHub's own org is
lower risk, and plenty of projects leave it on major tags, but a uniform rule is
the one people apply correctly a year later.

Two mechanics worth knowing:

- Some tags, such as `mozilla-actions/sccache-action@v0.0.11`, are *annotated*
  tags. `git ls-remote` and the refs API return the tag object's SHA, not the
  commit's. Pinning that SHA fails; peel it to the commit first
  (`gh api repos/OWNER/REPO/git/tags/<sha> -q .object.sha`).
- A trailing comment is not verified by anything. If you hand-edit a SHA, the
  comment can end up describing a different version than the pin.

### Why callers pin this repository to a tag

`@main` means every consumer picks up our changes on its next run, unreviewed
and unannounced. A caller that pinned a tag can review a bump; a caller on
`@main` finds out when a job fails.

```yaml
uses: rigetti/qcs-gha-infrastructure/.github/workflows/rust-ci.yml@v0.1.0
```

The usage examples above say `@main` for brevity. Real callers should not.

### Bumping this repository's own tag

The composite-action references *inside* these reusable workflows are pinned to
the matching release tag, so pinning a workflow pins what it uses transitively —
otherwise a caller would pin the outer layer while the action underneath kept
floating.

That makes a release a two-step change: update the internal references to the
new tag **in the same commit the tag will point at**, then tag that commit.
