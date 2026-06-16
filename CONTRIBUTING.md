# Contributing to LeavePulse

Thanks for your interest in LeavePulse. This guide applies across the
organization's public repositories.

## Before you start

Most of the platform is private while it stabilises; the public repositories are
the shared libraries and SDKs. A couple of them are **generated** and should not
be edited by hand:

- `leavepulse-sdk-ts`, `leavepulse-sdk-python`, and `leavepulse-sdk-rust` are
  generated from the platform's API and realtime contracts. Fixes belong in the
  generator, not in the published output — edits to generated files are
  overwritten on the next release.

When in doubt, open an issue first to check where a change should live.

## Reporting bugs and requesting features

- Search existing issues before opening a new one.
- For bugs, include the repository, version or commit, what you expected, what
  happened, and a minimal reproduction.
- For security issues, follow [SECURITY.md](SECURITY.md) instead of opening a
  public issue.

## Pull requests

1. Fork the repository and create a branch from the default branch.
2. Keep changes focused — one logical change per pull request.
3. Match the surrounding code style; do not reformat unrelated code.
4. Make sure the project's checks pass before opening the PR:
   - **Python** projects use [uv](https://docs.astral.sh/uv/) and
     [Ruff](https://docs.astral.sh/ruff/); run the project's test and lint
     commands (often surfaced through `lp-ci`).
   - **TypeScript / Vue** projects use [Bun](https://bun.sh); run
     `bun run typecheck` and any project lint/test scripts.
   - **Rust** projects should pass `cargo build`, `cargo clippy`, and
     `cargo test`.
5. Write a clear PR description explaining the change and its motivation.

## Commit messages

We follow [Conventional Commits](https://www.conventionalcommits.org/)
(`feat:`, `fix:`, `chore:`, `docs:`, etc.). Keep the subject concise and use the
body to explain the *why* when it isn't obvious.

## License

Unless a repository states otherwise, contributions are accepted under that
repository's license (MIT for the public libraries and SDKs). By submitting a
contribution, you agree to license it under the same terms.
