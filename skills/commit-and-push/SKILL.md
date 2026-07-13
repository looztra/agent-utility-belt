---
name: commit-and-push
description: Safely review git changes, create one or more semantic commits with mandatory scopes, and push the current branch. Use when the user asks to commit changes, push a branch, prepare commits for a PR, or perform a commit-and-push workflow with git.
---

- Check that we are in a branch that is not main or master. If the current branch is main or master, stop immediately and inform the user that commits should not be made directly to the protected branch.
- Check the list of changes using git commands.
- If there are uncommitted changes (staged or unstaged) detected in the previous step, commit them using semantic commit message conventions with a mandatory scope, and use the appropriate type for the changes made (fix, feat, refactor, chore, etc.).
- Use several commits if needed to properly describe the changes made according to their type.
- After committing, push the changes to the remote repository using git push, unless explicitly instructed not to do so. If git push fails because no upstream branch is set, use git push --set-upstream origin <current-branch-name> and report this action to the user. If git push fails for any other reason, report the exact error output to the user and do not attempt to force-push or alter remote state without explicit instruction.
