---
name: generate-pr
description: Create or update a GitHub pull request from the current branch with semantic commit conventions, safe commit selection, template-compliant PR descriptions, and tool fallback (GitHub CLI then GitHub MCP server).
argument-hint: Branch or change summary to use for the PR
---

# Generate Or Update Pull Request

## Outcome

Create a PR when none exists for the current branch, or update the existing PR title/description so it accurately represents all branch changes.

## When To Use

- Opening a PR for local branch changes
- Syncing an existing PR title/description after new commits
- Preparing a standards-compliant PR body from a repository template

## Procedure

1. Inspect repository state.
2. Determine the current branch and default branch.
3. If current branch equals default branch, stop and tell the user a feature branch is required before creating a PR.
4. Review local changes using git status/diff.
5. If there are uncommitted changes, apply commit rules:
  - Commit only files that are explicitly staged or clearly related to the feature/fix being PR'd.
  - Never commit unrelated work-in-progress, ignored files, secrets, or ambiguous files.
  - If inclusion is ambiguous, ask the user which files to include.
  - Create one commit per logical change.
  - Each commit must still follow conventional commit type + mandatory scope (for example `feat(scope): ...`, `fix(scope): ...`).
6. Push the current branch to origin.
7. Check whether a PR already exists for the current branch.
8. If no PR exists, create one.
9. If a PR exists, update title and description only when needed.

## PR Creation And Update Rules

1. Prefer GitHub CLI when available/authenticated.
2. If GitHub CLI is unavailable, use GitHub MCP tools.
3. If neither GitHub CLI nor GitHub MCP tools is available/authenticated, stop and instruct the user to run `gh auth login` (and install `gh` if missing) before retrying.
4. Target the repository default branch unless branch naming or existing PR metadata clearly indicates a different long-lived target (for example `release/*`).
5. For ambiguous base branch selection, ask the user before creating the PR.
6. PR title must follow semantic commit style with mandatory scope and describe the main change in the full PR, not just the last commit.
7. Update existing PR title/description when they do not reflect all branch commits, miss required template sections, or violate title convention.

## PR Description Rules

1. Use the repository PR template when one exists.
2. If no template is found, use this fallback structure: [Summary], [Changes], [Testing instructions].
3. Summarize all changes in the branch, not just the latest commit.
4. Group changes by type instead of listing each file.
5. Be concise and specific.
6. If required template fields need unknown information (for example related issue), stop and ask the user for the missing information instead of guessing.
7. In testing sections, include only manual/functional checks a reviewer can execute.
8. Do not include praise about CI/CD status, test counts, or coverage percentages.

## Accuracy And Safety Checks

- Never mark checkboxes for tasks not performed or verified by the current agent run.
- When updating an existing PR authored by others, preserve unchecked/checked state unless there is direct evidence the task was completed.
- Ensure the final PR text is consistent with branch contents and commit history.

## Important Context

The agent may not be the original author of branch commits. It is still responsible for delivering a complete, accurate PR title and description aligned with repository standards.
