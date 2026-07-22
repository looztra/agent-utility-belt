---
name: pr-merge-loop
description: Work through every open PR on the current repository, oldest first, rebasing and auto-merging approved PRs while skipping unapproved ones, then report a summary table of merged/not-merged PRs with reasons. Use when the user asks to "merge all approved PRs", "clear the PR queue", "rebase and merge open PRs", or run a merge loop/babysit open PRs until they're all resolved.
license: Apache-2.0
metadata:
  last-updated: 2026-07-22
---

# PR Merge Loop

## Outcome

Every open PR on the current repository has been either merged, or left open with a clear,
reported reason — and the user has a single summary table covering all of them.

## When To Use

- "Merge all the approved PRs"
- "Go through the open PRs and merge what's ready"
- "Babysit the PR queue until everything's merged"
- Any request to loop over open PRs, rebase them onto the base branch, and merge the approved ones

## Scope

- Always operates on the repository inferred from the current directory's git remote /
  `gh repo view` — never accept or infer a different `owner/repo` target.
- Only processes PRs open at the moment the run starts (the "worklist"). PRs opened by someone
  else after the run begins are out of scope for that run; mention them in the closing note if
  seen during the final re-check, but do not act on them.

## Preconditions

1. Verify `gh --version` and `gh auth status`. If either fails, stop and tell the user to install
   `gh` / run `gh auth login`.
2. Check the `gh` version is at least **v2.55.0** — earlier versions lack `gh pr update-branch
   --rebase`, which this skill depends on. If it is older, stop and tell the user to upgrade `gh`.
3. Verify the current directory is a GitHub-backed git repository: `gh repo view --json
   nameWithOwner`. If this fails, stop and ask the user which repository/directory to use.
4. Determine the allowed merge method once, in priority order squash > rebase > merge commit,
   via `gh repo view --json squashMergeAllowed,rebaseMergeAllowed,mergeCommitAllowed`. Use this
   method for every auto-merge enabled during the run.
5. Set a total run budget (default ~45 minutes) before starting. See the timeout handling in the
   Procedure.

## Procedure

1. Snapshot the worklist: `gh pr list --json number,title,createdAt,url --state open --limit 30`,
   sorted ascending by `createdAt` (oldest first). This list is fixed for the run — do not
   re-query it mid-loop. If exactly 30 PRs come back the list may be truncated; say so in the
   closing note.
2. Initialize an empty results table (columns: PR, Title, Merged?, Reason).
3. Because PRs are merged one at a time, every PR after the first will usually fall behind the
   base branch and need a rebase — which restarts its CI. Expect roughly one full CI run per PR,
   and let that inform how much of the run budget is left.
4. For each PR in the worklist, in order, do not start the next PR until the current one reaches
   one of the terminal outcomes below:
   a. Fetch state: `gh pr view <n> --json isDraft,reviewDecision,mergeStateStatus,mergeable,state,statusCheckRollup,autoMergeRequest`.
   b. **Draft** (`isDraft` is true): record `{Merged: No, Reason: "Draft"}` and move on — a draft
      cannot be merged even when approved.
   c. **Not approved** (`reviewDecision` is not `APPROVED`): tell the user this PR is not approved,
      take no further action on it, record `{Merged: No, Reason: "Not approved"}`, and move on to
      the next PR immediately — do not wait or poll for this one.
   d. **Mergeability not yet computed** (`mergeable` or `mergeStateStatus` is `UNKNOWN`): GitHub
      computes this asynchronously. Wait a few seconds and re-run step (a) — up to 3 times before
      treating it as a timeout. Never act on an `UNKNOWN` value.
   e. **Conflicting** (`mergeable == CONFLICTING`): record `{Merged: No, Reason: "Conflicts with
      base branch"}` and move on. Do not enable auto-merge and do not attempt to resolve the
      conflict.
   f. **Approved and mergeable**:
      - If `mergeStateStatus` indicates the branch is behind the base branch (e.g. `BEHIND`), run
        `gh pr update-branch <n> --rebase` to bring it up to date via rebase.
      - If auto-merge is not already enabled (`autoMergeRequest` is null), enable it with the merge
        method determined in Preconditions, e.g. `gh pr merge <n> --auto --squash`. Never enable
        auto-merge on a PR that isn't approved.
      - If that command fails because auto-merge is disabled on the repository, fall back to
        waiting for checks as below and then merging directly with the same method (e.g. `gh pr
        merge <n> --squash`) once — and only once — every check has passed. Still approved-only,
        still no bypass flags. Any other failure is handled by the Rules below.
      - Wait for checks with `gh pr checks <n> --watch --fail-fast --interval 30`, which blocks
        until the checks settle and exits non-zero on failure. If that command is unavailable,
        fall back to polling `gh pr view <n> --json state,statusCheckRollup` every ~30s. Either
        way, cap this PR at ~10 minutes and stop at one of:
        - `state == MERGED` → record `{Merged: Yes}`. This PR is done; continue to the next PR.
        - A failing check in `statusCheckRollup` → record `{Merged: No, Reason: "Checks failed:
          <failing check name(s)>"}`. The rollup mixes two node types: check runs report
          `conclusion` (`FAILURE`, `CANCELLED`, `TIMED_OUT`, `ACTION_REQUIRED`, `STARTUP_FAILURE`)
          and legacy status contexts report `state` (`FAILURE`, `ERROR`) — inspect both fields.
          `NEUTRAL` and `SKIPPED` are not failures. Do **not** retry automatically and do not stop
          the whole run — this PR simply cannot be merged right now, but it must still show up in
          the final summary table. Continue to the next PR.
        - `state == CLOSED` without merging → record `{Merged: No, Reason: "Closed without
          merging"}`. Continue to the next PR.
        - Timeout reached while still pending → record `{Merged: No, Reason: "Timed out waiting
          on checks"}`. Continue to the next PR.
   g. If the total run budget from Preconditions is exhausted, stop enabling auto-merge on any
      further PR. Record each remaining worklist entry as `{Merged: No, Reason: "Not reached —
      run budget exhausted"}` and go straight to the summary table — the table is always
      produced.
5. Once every PR in the worklist has a terminal outcome, print the summary table (see Output
   Format).
6. Re-list open PRs (`gh pr list --state open --limit 30`) once. If any PR from the worklist is still open
   (not-approved, checks-failed, closed-without-merging, or timed-out cases), or if new PRs opened
   during the run are now visible, mention this briefly below the table — but take no action on
   PRs outside the original worklist.

## Rules

- Never force-merge, never bypass branch protection, required reviews, or required checks.
- Never enable auto-merge on a PR that is not approved.
- Never retry a genuine CI failure automatically — a human should look at it.
- Never rebase/force-push to a PR branch other than via `gh pr update-branch --rebase`.
- If `gh pr update-branch` or `gh pr merge --auto` fails outright (e.g. permissions, conflicts),
  record the exact error as the `Reason` for that PR and move on — do not attempt to resolve
  conflicts yourself. The single exception is auto-merge being disabled repository-wide, which
  has an explicit fallback in the Procedure.
- Always produce the summary table, even when the run is cut short by the budget or an error.

## Output Format

Always end with a Markdown table, oldest PR first, one row per PR in the worklist:

| PR | Title | Merged? | Reason |
| -- | ----- | ------- | ------ |
| #14 | chore(deps): update zizmor-action to v0.6.0 | Yes | — |
| #18 | chore(deps): update actionlint action to v1.73.0 | No | Checks failed: Pre-commit |

Follow the table with at most one short sentence of closing context (e.g. new PRs seen on
re-check). Do not repeat per-PR narration that already happened during the loop.
