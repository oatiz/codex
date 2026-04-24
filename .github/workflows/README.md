# Workflow Strategy

The workflows in this directory are split so that pull requests get fast, review-friendly signal while `main` still gets the full cross-platform verification pass.

## Pull Requests

- `bazel.yml` is the main pre-merge verification path for Rust code.
  It runs Bazel `test` and Bazel `clippy` on the supported Bazel targets,
  including the generated Rust test binaries needed to lint inline `#[cfg(test)]`
  code.
- `rust-ci.yml` keeps the Cargo-native PR checks intentionally small:
  - `cargo fmt --check`
  - `cargo shear`
  - `argument-comment-lint` on Linux, macOS, and Windows
  - `tools/argument-comment-lint` package tests when the lint or its workflow wiring changes

## Post-Merge On `main`

- `bazel.yml` also runs on pushes to `main`.
  This re-verifies the merged Bazel path and helps keep the BuildBuddy caches warm.
- `rust-ci-full.yml` is the full Cargo-native verification workflow.
  It keeps the heavier checks off the PR path while still validating them after merge:
  - the full Cargo `clippy` matrix
  - the full Cargo `nextest` matrix
  - release-profile Cargo builds
  - cross-platform `argument-comment-lint`
  - Linux remote-env tests

## Releases

- `rust-release.yml` is the **official release workflow**.
  It is triggered by `rust-v*` tags and handles code signing, notarization, npm
  packaging, installer scripts, DotSlash publication, and WinGet submission.
  All public distribution channels (npm, installer scripts, WinGet, GitHub
  Releases marked as `latest`) flow through this workflow.
- `codex-bin-release.yml` is a **lightweight binary-only release workflow**.
  It is triggered manually via `workflow_dispatch` and builds only the
  standalone `codex` CLI binary for Linux (x64, ARM64), macOS (Intel, Apple
  Silicon), and Windows (x64) using GitHub-hosted runners.  It is safe to run
  on forks and intentionally does not touch npm, installers, code signing, or
  any other official distribution path.
  - Set the `program_name` input to customize the emitted binary name, CLI help
    text, and default config home directory.  The customization is applied as a
    transient CI-only patch and is never committed.
  - Set `publish_release` to create a prerelease under the `codex-bin-v*` tag
    namespace.  These prereleases are always marked as prerelease and never
    update the repository's `latest` release pointer.

## Rule Of Thumb

- If a build/test/clippy check can be expressed in Bazel, prefer putting the PR-time version in `bazel.yml`.
- Keep `rust-ci.yml` fast enough that it usually does not dominate PR latency.
- Reserve `rust-ci-full.yml` for heavyweight Cargo-native coverage that Bazel does not replace yet.
- Use `rust-release.yml` (via tag push) for official public releases.
- Use `codex-bin-release.yml` (via manual dispatch) to produce a quick standalone binary without the overhead of the official release pipeline.
