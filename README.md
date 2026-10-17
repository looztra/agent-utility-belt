# agent-utility-belt

My agents' go-to belt.

## Available skills

| Skill | Description |
| --- | --- |
| [adversarial-review](skills/adversarial-review/SKILL.md) | Run an adversarial review of a diff, branch, or PR using independent reviewer lenses and a synthesized verdict, optionally posting the findings as a pending GitHub PR review. |
| [commit-and-push](skills/commit-and-push/SKILL.md) | Safely review git changes, create one or more semantic commits with mandatory scopes, and push the current branch. |
| [document-architecture](skills/document-architecture/SKILL.md) | Analyze an existing application and create or update a concise, evidence-based Markdown architecture document set, clearly separating confirmed facts from unknowns. |
| [generate-pr](skills/generate-pr/SKILL.md) | Create or update a GitHub pull request from the current branch, with template-compliant descriptions and a GitHub CLI → GitHub MCP fallback. |
| [pr-merge-loop](skills/pr-merge-loop/SKILL.md) | Work through every open PR oldest first, rebasing and auto-merging the approved ones, then report a summary table of what merged and why the rest did not. |

## Installing the skills

### Using `npx skills` (recommended)

Install with [vercel-labs/skills](https://github.com/vercel-labs/skills), which fetches `SKILL.md` files straight from this GitHub repo:

```bash
# list what's available in this repo
npx skills add looztra/agent-utility-belt --list

# install one skill
npx skills add looztra/agent-utility-belt --skill adversarial-review --agent universal
npx skills add looztra/agent-utility-belt --skill commit-and-push --agent universal
npx skills add looztra/agent-utility-belt --skill document-architecture --agent universal
npx skills add looztra/agent-utility-belt --skill generate-pr --agent universal
npx skills add looztra/agent-utility-belt --skill pr-merge-loop --agent universal

# install every skill in the repo in universal mode
npx skills add looztra/agent-utility-belt --skill '*' --agent universal
```

`--agent universal` installs into the agent-agnostic `.agents/skills/` directory of the current project. Add `--global` to install at the user level (`~/.agents/skills/`) instead, or use `--agent <agent>` to target a specific agent. Do not use `--all` unless you want every skill installed to every supported agent; upstream defines it as shorthand for all skills plus all agents without prompts.

### Using the GitHub CLI (`gh`)

GitHub CLI v2.90.0+ ships a native [`gh skill`](https://cli.github.com/manual/gh_skill) command group (currently in preview) that discovers and installs skills directly from a repository — no manual clone needed:

```bash
# check your gh version supports it (needs v2.90.0+)
gh --version

# install one skill in universal mode (.agents/skills)
gh skill install looztra/agent-utility-belt adversarial-review --agent universal
gh skill install looztra/agent-utility-belt commit-and-push --agent universal
gh skill install looztra/agent-utility-belt document-architecture --agent universal
gh skill install looztra/agent-utility-belt generate-pr --agent universal
gh skill install looztra/agent-utility-belt pr-merge-loop --agent universal

# or install every skill in the repo
gh skill install looztra/agent-utility-belt --all --agent universal
```

Add `--scope user` to install at the user level instead of the current project (the default), or omit `--agent` to be prompted interactively. See `gh skill install --help` for pinning to a specific tag/commit and other options.
