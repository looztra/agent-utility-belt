---
name: adversarial-review
description: Adversarial code review that spawns independent reviewers to challenge the work from distinct critical lenses (Skeptic, Architect, Minimalist) and synthesizes a single verdict with findings and a lead judgment. Read-only — it never modifies code. Use when the user asks for an "adversarial review", a "hostile review", to "challenge this diff/plan", or to stress-test a change before merge.
argument-hint: "[what to review]"
license: Apache-2.0
metadata:
  last-updated: 2026-07-24
---

# Adversarial Review

Spawn independent reviewers to challenge work from distinct critical lenses. The deliverable
is a synthesized verdict — do **NOT** make changes to the code under review.

Model choice is the user's responsibility: run the reviewers on whatever model the current
session uses, unless the user explicitly names a model for the reviewers — in that case pass
it through as the reviewers' model. Do not detect, infer, or enforce any particular model.

## Step 1 — Load Grounding

Read the project's agent instructions (`CLAUDE.md`, `AGENTS.md`, and any documented
engineering principles they reference). These govern reviewer judgments: findings should be
anchored in the project's own conventions and the lens principles below, not personal taste.

### Spec-driven development (SDD) frameworks

Check whether the repo under review uses an SDD framework — if it does, its spec artifacts
are the authoritative statement of intent and acceptance criteria and MUST be loaded:

| Framework | Detection markers | Artifacts to load |
| --------- | ----------------- | ----------------- |
| BMAD Method | `.bmad-core/`, `_bmad/`, or `bmad/` directory | The active story under `docs/stories/`, plus the `docs/prd.md` and `docs/architecture.md` sections it references |
| OpenSpec | `openspec/` directory containing `project.md` | `openspec/project.md` and the active change under `openspec/changes/<id>/` (`proposal.md`, `tasks.md`, `design.md`, spec deltas) |
| Spec Kit | `.specify/` directory | `.specify/memory/constitution.md` and the active feature under `specs/<NNN-name>/` (`spec.md`, `plan.md`, `tasks.md`) |

Identify the *active* artifact (story / change / feature) from the branch name, recent
commits, or the files touched by the diff; if several are plausible, ask the user rather
than guessing. If markers are present but no active artifact can be tied to the work under
review, say so in the verdict — reviewing spec-managed work without its spec is itself
worth flagging.

## Step 2 — Determine Scope and Intent

Identify what to review from context (recent diffs, referenced plans, the user's message).
If nothing is identifiable, ask the user instead of guessing.

Determine the **intent** — what the author is trying to achieve. This is critical: reviewers
challenge whether the work *achieves the intent well*, not whether the intent is correct.
State the intent explicitly before proceeding.

If an SDD framework was detected in Step 1, derive the intent from the active spec artifact
(story, change proposal, or feature spec) rather than reconstructing it from the diff, and
treat its acceptance criteria / requirements as part of the intent. Divergence between the
implementation and its spec — unmet criteria, silent scope creep, behavior the spec never
asked for — is a finding, not a stylistic remark.

Assess change size:

| Size | Threshold | Reviewers |
| ---- | --------- | --------- |
| Small | < 50 lines, 1-2 files | 1 (Skeptic) |
| Medium | 50-200 lines, 3-5 files | 2 (Skeptic + Architect) |
| Large | 200+ lines or 5+ files | 3 (Skeptic + Architect + Minimalist) |

## Step 3 — Spawn Reviewers

Spawn one subagent per lens, all in parallel, read-only (no file edits, no commits). Each
reviewer gets a single self-contained prompt containing:

1. The stated intent (from Step 2)
2. Their assigned lens — the full lens text from the Reviewer Lenses section below
3. The relevant project conventions from Step 1 (contents, not summaries), including the
   active SDD spec artifacts when a framework was detected
4. The code or diff to review (or precise instructions to read it)
5. These instructions, verbatim: "You are an adversarial reviewer. Your job is to find real
   problems, not validate the work. Be specific — cite files, lines, and concrete failure
   scenarios. Rate each finding: high (blocks ship), medium (should fix), low (worth noting).
   Return your findings as a numbered markdown list."

If a reviewer fails or returns nothing, note the failure in the verdict — do not silently
skip a lens.

## Step 4 — Synthesize Verdict

Read each reviewer's findings. Deduplicate overlapping findings, keeping the highest severity
and crediting every lens that raised it. Produce a single verdict in the Verdict Format below.

## Step 5 — Render Judgment

After synthesizing the reviewers, apply your own judgment. Using the stated intent and the
project conventions as your frame, state which findings you would accept and which you would
reject — and why. Reviewers are adversarial by design; not every finding warrants action.
Call out false positives, overreach, and findings that mistake style for substance.

Append the Lead Judgment section to the verdict. Then stop — applying fixes is a separate
task the user must ask for.

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
<for each finding: accept or reject with a one-line rationale>
```

Verdict logic:

- **PASS** — no high-severity findings
- **CONTESTED** — high-severity findings but reviewers disagree on them
- **REJECT** — high-severity findings with reviewer consensus
