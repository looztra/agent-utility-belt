---
name: adversarial-review
description: Adversarial code review that spawns independent reviewers to challenge a change from distinct critical lenses (Skeptic, Architect, Minimalist) and synthesizes a single verdict with findings and a lead judgment. Works on uncommitted work, a branch diff, or a pull request — whether the current session's own or another author's. Read-only — it never modifies code and never posts review comments. Use when the user asks for an "adversarial review", a "hostile review", to "challenge this diff/plan", or to stress-test a change or PR.
argument-hint: "[what to review — diff, branch, or PR number]"
license: Apache-2.0
metadata:
  last-updated: 2026-07-25
---

# Adversarial Review

Spawn independent reviewers to challenge work from distinct critical lenses. The deliverable is
a synthesized verdict reported back to the user — do **NOT** make changes to the code under
review, and do **NOT** post the verdict as PR review comments unless the user explicitly asks.

The work under review may be the current session's own, or a pull request opened by someone
else. Both are in scope.

Model choice is the user's responsibility: run the reviewers on whatever model the current
session uses, unless the user explicitly names a model for the reviewers. If the agent runtime
supports per-reviewer model selection, pass through the requested model; otherwise state that
model selection is not controllable in the current environment. Do not detect, infer, or enforce
any particular model.

## Step 1 — Load Grounding

Read the project's agent instructions. These govern reviewer judgments: findings should be
anchored in the project's own conventions and the lens principles below, not personal taste.
Load whichever of these exist — they coexist rather than supersede one another, and a repo
written against Copilot may carry no `CLAUDE.md` at all:

| Source | Notes |
| ------ | ----- |
| `CLAUDE.md` | Repo root, plus any nested `CLAUDE.md` in directories the diff touches |
| `AGENTS.md` | Same nesting convention; read by several agent tools, Copilot among them |
| `.github/copilot-instructions.md` | Repo-wide Copilot instructions, applying to every file |
| `.github/instructions/*.instructions.md` | Path-scoped Copilot instructions. Each carries an `applyTo` frontmatter glob — load only those whose glob matches files in the diff, and pass each reviewer the ones matching the files it is judging |

Follow `@`-style imports and any documented engineering principles these files reference. Where
two sources genuinely conflict, say so in the verdict instead of silently picking one — a
contradiction between a repo's own instruction files is worth surfacing.

### Spec-driven development (SDD) frameworks

Check whether the repo under review uses an SDD framework — if it does, its spec artifacts
are the authoritative statement of intent and acceptance criteria and MUST be loaded:

| Framework | Detection markers | Artifacts to load |
| --------- | ----------------- | ----------------- |
| OpenSpec | `openspec/` directory containing `project.md` | `openspec/project.md` and the active change under `openspec/changes/<id>/` (`proposal.md`, `tasks.md`, `design.md`, spec deltas) |
| Spec Kit | `.specify/` directory | `.specify/memory/constitution.md` and the active feature under `specs/<NNN-name>/` (`spec.md`, `plan.md`, `tasks.md`) |

Identify the *active* artifact (change / feature) from the branch name, recent commits, the PR
description, or the files touched by the diff; if several are plausible, ask the user rather
than guessing. If markers are present but no active artifact can be tied to the work under
review, say so in the verdict — reviewing spec-managed work without its spec is itself
worth flagging.

BMAD Method is deliberately not covered: it ships its own review workflow, so this skill adds
little there. If BMAD markers are present (`_bmad/`, `_bmad-output/`, `.bmad-core/`), say so and
point the user at BMAD's own review before going further. If they still want an adversarial
review, proceed treating the repo as *No SDD framework* below — do not attempt to locate or load
BMAD artifacts.

### No SDD framework

Most repos have no spec framework. This is the **default case, not a degraded one** — but it
changes what reviewers can be held to, because there is no authoritative statement of intent to
check the implementation against. Two adjustments:

- Weight the project's own conventions more heavily, since they are the only written standard
  available: every agent instruction source loaded in Step 1, plus the existing patterns in the
  code surrounding the change.
- The intent is a *reconstruction* (see Step 2). Tell reviewers so explicitly. A finding that
  rests on inferred intent is weaker than one resting on a written requirement, and reviewers
  that treat their own reconstruction as authoritative produce confident nonsense.

## Step 2 — Determine Scope and Intent

Identify what to review from context (the user's message, recent diffs, a referenced PR or plan).
If nothing is identifiable, ask the user instead of guessing.

| Target | How to obtain it |
| ------ | ---------------- |
| Uncommitted work | `git diff` and `git diff --staged` |
| A branch | `git diff <base>...HEAD` — three-dot, against the merge base, so commits landed on the base branch since do not appear as part of the change |
| A pull request | `gh pr view <n> --json author,body,comments,commits,reviews,title,url` for metadata and top-level discussion; `gh api repos/{owner}/{repo}/pulls/<n>/comments` for inline review comments (`{owner}`/`{repo}` are expanded by `gh` from the current repo remote — use literal `owner/repo` if not in a repo context); `gh pr diff <n>` for the change |

### Reviewing someone else's pull request

A PR opened by another author is a first-class target — this skill is not limited to the current
session's own work, nor to work that has yet to be submitted. When the author is someone else:

- Take intent from what the PR actually claims (description, linked issue, commit messages), not
  from what you would have built. Reconstructing a different intent and then reviewing against it
  is the dominant failure mode here, and it produces findings the author will rightly dismiss.
- Read the existing review comments before spawning reviewers, and drop findings already raised.
  Restating another reviewer's point as a fresh discovery is noise. Pass a concise "known prior
  findings" block to each reviewer so they do not spend their pass rediscovering already-raised
  issues.
- The branch may not be checked out. `gh pr diff` covers the diff, but reviewers needing
  surrounding context must read files from the PR head without disturbing the current worktree
  when possible: fetch the PR head ref and inspect files with `git show <ref>:<path>`, or use a
  temporary worktree with explicit user consent. If neither is possible, state plainly in the
  verdict which context could not be inspected.

### Intent

Determine the **intent** — what the author is trying to achieve. This is critical: reviewers
challenge whether the work *achieves the intent well*, not whether the intent is correct. State
the intent explicitly before proceeding, along with where it came from, in this order of
authority:

1. **The active SDD spec artifact** from Step 1, when one exists. Its acceptance criteria and
   requirements are part of the intent. Divergence between implementation and spec — unmet
   criteria, silent scope creep, behavior the spec never asked for — is a finding, not a
   stylistic remark.
2. **The PR description and any linked issue.**
3. **Commit messages and the diff itself** — a reconstruction. Label it as such in the reviewer
   prompts and in the verdict, so a reader can tell which findings rest on a stated requirement
   and which rest on an inference.

Assess change size. Classify by the **highest tier any single dimension reaches** — e.g. a 100-line, 1-file change is Medium (lines trigger Medium, files only trigger Small, take the higher):

| Size | Threshold | Reviewers |
| ---- | --------- | --------- |
| Small | < 50 lines **and** 1-2 files | 1 (Skeptic) |
| Medium | 50-200 lines **or** 3-5 files | 2 (Skeptic + Architect) |
| Large | 200+ lines **or** 5+ files | 3 (Skeptic + Architect + Minimalist) |

## Step 3 — Spawn Reviewers

Spawn one subagent per lens, all in parallel, read-only (no file edits, no commits). Each
reviewer gets a single self-contained prompt containing:

1. The stated intent (from Step 2) **and its source** — spec artifact, PR description, or
   reconstruction from the diff
2. Their assigned lens — the full lens text from the Reviewer Lenses section below
3. The relevant project conventions from Step 1 (contents, not summaries), including the
   active SDD spec artifacts when a framework was detected
4. Known prior findings from existing PR comments, when reviewing a PR, so reviewers avoid
   duplicate reports
5. The code or diff to review (or precise instructions to read it)
6. These instructions, verbatim: "You are an adversarial reviewer. Your job is to find real
   problems, not validate the work. Be specific — cite files, lines, and concrete failure
   scenarios. Rate each finding: high (blocks ship), medium (should fix), low (worth noting).
   Return your findings as a numbered markdown list."

If a reviewer fails or returns nothing, note the failure in the verdict — do not silently
skip a lens.

## Step 4 — Synthesize Verdict

Read each reviewer's findings. Deduplicate overlapping findings, keeping the highest severity
and crediting every lens that raised it. Do not treat lack of overlap as evidence that a finding
is weak: each reviewer used a different lens, so a valid issue may appear in only one review.

## Step 5 — Render Judgment

After synthesizing the reviewers, apply your own judgment before assigning the verdict. Using the
stated intent and the project conventions as your frame, state which findings you would accept,
reject, or mark uncertain — and why. Reviewers are adversarial by design; not every finding
warrants action. Call out false positives, overreach, and findings that mistake style for
substance.

Assign the final verdict from the accepted and uncertain findings only, then append the Lead
Judgment section. Rejected findings do not affect the verdict. Then stop — applying fixes is a
separate task the user must ask for.

## Reviewer Lenses

Three distinct adversarial perspectives. Each reviewer adopts one lens exclusively.

### Skeptic

Challenge correctness and completeness. Ask:

- What inputs, states, or sequences will break this?
- What error paths are unhandled or silently swallowed?
- What race conditions or ordering dependencies exist?
- What does the author believe is true that isn't proven?
- Where is "it works on my machine" masquerading as verification?

### Architect

Challenge structural fitness. Ask:

- Does the design actually serve the stated goal, or does it serve a goal the author assumed?
- Where are the coupling points that will hurt when requirements shift?
- What boundary violations exist? Where does responsibility leak between components?
- What implicit assumptions about scale, concurrency, or ordering will break first?

### Minimalist

Challenge necessity and complexity. Ask:

- What can be deleted without losing the stated goal?
- Where is the author solving problems they don't have yet?
- What abstractions exist for a single call site?
- Where is configuration or flexibility added without a concrete second use case?
- Is this the simplest possible path to the outcome, or is it the path that felt most thorough?

## Verdict Format

```markdown
## Intent
<what the author is trying to achieve>
<source: spec artifact / PR description + issue / reconstructed from the diff>

## Verdict: PASS | CONTESTED | REJECT
<one-line summary>

## Findings
<numbered list, ordered by severity (high -> medium -> low)>

For each finding:
- **[severity]** Description with file:line references
- Lens: which reviewer raised it
- Recommendation: concrete action, not vague advice

## What Went Well
<1-3 things the reviewers found no issue with — acknowledge good work>

## Lead Judgment
<for each finding: accept, reject, or uncertain with a one-line rationale>
```

Verdict logic (findings marked uncertain by the Lead Judgment that are not high-severity are treated as effectively accepted-low: they appear in the Lead Judgment but do not push toward CONTESTED):

- **PASS** — no accepted medium- or high-severity findings, and no uncertain high-severity
  findings
- **CONTESTED** — at least one uncertain high-severity finding, or accepted medium-severity
  findings without accepted high-severity findings
- **REJECT** — at least one accepted high-severity finding
