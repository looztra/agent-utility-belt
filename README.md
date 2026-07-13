# agent-utility-belt

My agents' go-to belt.

## Available skills

| Skill | Description |
| --- | --- |
| [commit-and-push](skills/commit-and-push/SKILL.md) | Safely review git changes, create one or more semantic commits with mandatory scopes, and push the current branch. |
| [generate-pr](skills/generate-pr/SKILL.md) | Create or update a GitHub pull request from the current branch, with template-compliant descriptions and a GitHub CLI → GitHub MCP fallback. |

## Installing the skills

### Using `npx skills` (recommended)

Install with [vercel-labs/skills](https://github.com/vercel-labs/skills), which fetches `SKILL.md` files straight from this GitHub repo:

```bash
# list what's available in this repo
npx skills add looztra/agent-utility-belt --list

# install specific skills
npx skills add looztra/agent-utility-belt --skill commit-and-push --skill generate-pr

# or install every skill in the repo
npx skills add looztra/agent-utility-belt --all
```

Add `-g` to install at the user level (`~/.claude/skills/`) instead of the current project, or `-a <agent>` to target an agent other than the auto-detected one.

### Using the GitHub CLI (`gh`)

GitHub CLI v2.90.0+ ships a native [`gh skill`](https://cli.github.com/manual/gh_skill) command group (currently in preview) that discovers and installs skills directly from a repository — no manual clone needed:

```bash
# check your gh version supports it (needs v2.90.0+)
gh --version

# install a specific skill for Claude Code
gh skill install looztra/agent-utility-belt commit-and-push --agent claude-code
gh skill install looztra/agent-utility-belt generate-pr --agent claude-code

# or install every skill in the repo
gh skill install looztra/agent-utility-belt --all --agent claude-code
```

Add `--scope user` to install at the user level instead of the current project (the default), or omit `--agent` to be prompted interactively. See `gh skill install --help` for pinning to a specific tag/commit and other options.
