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
| `push` to the release branch | full release: bump, changelog, commit, tag, push |

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

## Versioning

Callers may pin to a tag (`@v0.1.0`) instead of `@main` once tags exist. The
composite-action references inside the reusable workflows are pinned to `main`;
a caller pinning a workflow tag does not pin those transitively.
