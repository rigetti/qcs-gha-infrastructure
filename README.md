# qcs-gha-infrastructure

Shared GitHub Actions and reusable workflows for QCS repositories. This is the
GitHub Actions counterpart to the GitLab CI templates in
`rigetti/qcs/utilities/qcs-infrastructure`.

## Composite actions

### `actions/setup-rust`

Installs a Rust toolchain with rustup and restores the cargo/target cache.

```yaml
- uses: rigetti/qcs-gha-infrastructure/actions/setup-rust@main
  with:
    toolchain: stable        # optional
    components: clippy       # optional, space-separated
    targets: x86_64-unknown-linux-musl  # optional, space-separated
    cache: "true"            # optional
    cache-key: clippy        # optional cache discriminator
```

### `actions/setup-knope`

Installs the [knope](https://knope.tech) release-automation CLI.

```yaml
- uses: rigetti/qcs-gha-infrastructure/actions/setup-knope@main
  with:
    version: "0.23.0"  # optional
```

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

### `knope-dry-run.yml`

Verifies on pull requests that a release can be prepared.

```yaml
jobs:
  knope-dry-run:
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/knope-dry-run.yml@main
```

### `knope-release.yml`

Runs knope's `release` workflow: bump versions, update the changelog, commit,
tag, and push.

```yaml
jobs:
  release:
    uses: rigetti/qcs-gha-infrastructure/.github/workflows/knope-release.yml@main
    secrets:
      token: ${{ secrets.RELEASE_TOKEN }}
```

Note: pushes made with the default `GITHUB_TOKEN` do not trigger further
workflow runs. If the release commit or tag must trigger a build-and-publish
workflow, pass a PAT or GitHub App token instead.

## Versioning

Callers may pin to a tag (`@v0.1.0`) instead of `@main` once tags exist. The
composite-action references inside the reusable workflows are pinned to `main`;
a caller pinning a workflow tag does not pin those transitively.
