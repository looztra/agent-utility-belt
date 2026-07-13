# AGENTS.md

Instructions for AI coding agents working in this repository.

## What this repo is

`agent-utility-belt` is a collection of Claude Code skills (see [skills/](skills/)), each a
self-contained `SKILL.md` describing a workflow (git, PR management, etc.). There is no
application code to build or run — changes are almost always edits to skill markdown files,
repo tooling config, or GitHub workflows.

## Commits

- Every commit must follow [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)
  **with a mandatory scope**: `type(scope): summary` (for example `feat(skills): add foo skill`,
  `fix(precommit): correct taplo entry`). A commit without a scope is not acceptable here.
- Use the type that matches the change (`feat`, `fix`, `refactor`, `chore`, `docs`, `ci`, etc.) —
  commit types drive the release-please changelog (see [release-please-config.json](release-please-config.json)).
- Prefer one commit per logical change rather than a single catch-all commit.

## Pull requests

- PR titles are linted in CI (`.github/workflows/lint-pr-title.yaml`) and must also follow
  Conventional Commits **with a scope** (`requireScope: true`) — a PR title without one will fail
  the check and block merge.
- The PR title should describe the overall change across all commits, not just the last commit.

## Pre-commit checks (must pass)

This repo enforces formatting/linting via [pre-commit](https://pre-commit.com/), configured in
[.pre-commit-config.yaml](.pre-commit-config.yaml) and run through [mise](https://mise.jdx.dev/)
tasks defined in [toolbox/mise/tasks/toml/](toolbox/mise/tasks/toml/):

```bash
mise run precommit:install   # one-time: install the git hook
mise run precommit:run       # run all hooks against all files
```

Run `mise run precommit:run` before committing/pushing and fix any reported issues — CI runs the
exact same command (`.github/workflows/code-checks.yaml`, job "Pre-commit") and the PR cannot merge
if it fails. The hook set includes, among others: trailing-whitespace/EOF fixers, YAML/TOML/JSON
validation, `markdownlint-cli2`, `shellcheck`/`shfmt`, `yamllint`, `yamkix`, and `taplo` formatting
for TOML files.

If touching `.github/workflows/`, also expect `actionlint` and `zizmor` checks (`mise run
check:actionlint`, `mise run check:zizmor`, also run in CI via `.github/workflows/workflows-checks.yaml`).

## Skills directory conventions

- Each skill lives in `skills/<name>/SKILL.md` with YAML frontmatter (`name`, `description`, and
  optionally `argument-hint`).
- Keep skills focused and avoid duplicating logic already covered by another skill — reference/delegate
  to the other skill instead of restating its rules (see how `generate-pr` delegates commit/push
  handling to `commit-and-push`).
