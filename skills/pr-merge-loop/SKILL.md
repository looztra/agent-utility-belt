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
2. Verify the current directory is a GitHub-backed git repository: `gh repo view --json
   nameWithOwner`. If this fails, stop and ask the user which repository/directory to use.
3. Determine the allowed merge method once, in priority order squash > rebase > merge commit,
   via `gh repo view --json squashMergeAllowed,rebaseMergeAllowed,mergeCommitAllowed`. Use this
   method for every auto-merge enabled during the run.

## Procedure

1. Snapshot the worklist: `gh pr list --json number,title,createdAt,url --state open`, sorted
   ascending by `createdAt` (oldest first). This list is fixed for the run — do not re-query it
   mid-loop.
2. Initialize an empty results table (columns: PR, Title, Merged?, Reason).
3. For each PR in the worklist, in order, do not start the next PR until the current one reaches
   one of the terminal outcomes below:
   a. Fetch state: `gh pr view <n> --json reviewDecision,mergeStateStatus,mergeable,state,statusCheckRollup,autoMergeRequest`.
   b. **Not approved** (`reviewDecision` is not `APPROVED`): tell the user this PR is not approved,
      take no further action on it, record `{Merged: No, Reason: "Not approved"}`, and move on to
      the next PR immediately — do not wait or poll for this one.
   c. **Approved**:
      - If `mergeStateStatus` indicates the branch is behind the base branch (e.g. `BEHIND`), run
        `gh pr update-branch <n> --rebase` to bring it up to date via rebase.
      - If auto-merge is not already enabled (`autoMergeRequest` is null), enable it with the merge
        method determined in Preconditions, e.g. `gh pr merge <n> --auto --squash`. Never enable
        auto-merge on a PR that isn't approved.
      - Poll `gh pr view <n> --json state,statusCheckRollup` on an interval (~20-30s, overall
        timeout ~10 minutes per PR) until one of:
        - `state == MERGED` → record `{Merged: Yes}`. This PR is done; continue to the next PR.
        - Any check in `statusCheckRollup` has concluded `FAILURE`, `CANCELLED`, or `TIMED_OUT` →
          record `{Merged: No, Reason: "Checks failed: <failing check name(s)>"}`. Do **not**
          retry automatically and do not stop the whole run — this PR simply cannot be merged
          right now, but it must still show up in the final summary table. Continue to the next PR.
        - `state == CLOSED` without merging → record `{Merged: No, Reason: "Closed without
          merging"}`. Continue to the next PR.
        - Timeout reached while still pending → record `{Merged: No, Reason: "Timed out waiting
          on checks"}`. Continue to the next PR.
4. Once every PR in the worklist has a terminal outcome, print the summary table (see Output
   Format).
5. Re-list open PRs (`gh pr list --state open`) once. If any PR from the worklist is still open
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
  conflicts yourself.

## Output Format

Always end with a Markdown table, oldest PR first, one row per PR in the worklist:

| PR | Title | Merged? | Reason |
| -- | ----- | ------- | ------ |
| #14 | chore(deps): update zizmor-action to v0.6.0 | Yes | — |
| #18 | chore(deps): update actionlint action to v1.73.0 | No | Checks failed: Pre-commit |

Follow the table with at most one short sentence of closing context (e.g. new PRs seen on
re-check). Do not repeat per-PR narration that already happened during the loop.
