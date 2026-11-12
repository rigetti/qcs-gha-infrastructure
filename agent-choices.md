# Agent choices

- Label: `macos-latest`, per the user's choice, over a pinned `macos-15`.
  It floats to newer images over time; consumers wanting stability can override `matrix`.
- Architecture: `macos-latest` is `macos-26` arm64 per the `actions/runner-images` README,
  so `aarch64-apple-darwin` stays a native build.
- Scope: every `macos-14` outside `CHANGELOG.md` (3 places).
  Nothing knope generates was touched.
- Commit type is `fix:`: only the default matrix changes.
  Consumers pinned to older tags are unaffected.
- Pin lint and actionlint concern `uses:` SHAs and expression syntax; neither depends on the runner label.
- Local checks: `yq` parses the edited workflows.
  `actionlint` is not installed locally; CI runs it.
- Delete this file before merging.
