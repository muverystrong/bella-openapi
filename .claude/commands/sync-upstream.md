---
description: Sync fork with upstream repository (LianjiaTech/bella-openapi)
allowed-tools: Bash, AskUserQuestion
---

Sync this fork with the upstream repository. Follow these steps:

## 1. Verify upstream remote

Run `git remote -v` to confirm `upstream` points to `https://github.com/LianjiaTech/bella-openapi.git`.

If `upstream` is not configured, add it:
```
git remote add upstream https://github.com/LianjiaTech/bella-openapi.git
```

## 2. Check current status

Run `git status` to check for uncommitted changes. If there are any, warn the user and ask whether to stash them first before proceeding.

## 3. Fetch upstream

```
git fetch upstream
```

Show the user how many new commits are coming in:
```
git log HEAD..upstream/main --oneline
```

If there are no new commits, inform the user the fork is already up to date and stop.

## 4. Ask the user for sync strategy

Ask the user which strategy to use:
- **merge** (default): `git merge upstream/main` — preserves full history, safer for shared branches
- **rebase**: `git rebase upstream/main` — linear history, cleaner but rewrites commits

## 5. Execute the sync

Based on their choice, run the appropriate command on the **current branch**.

If there are merge conflicts:
1. List the conflicting files
2. Ask the user how they want to resolve: manually, or abort and stop
3. If abort: run `git merge --abort` or `git rebase --abort`

## 6. Report result

After a successful sync:
- Show `git log --oneline -5` to confirm the result
- Remind the user to push to origin if needed: `git push origin <current-branch>`
  - If rebased, warn that force push (`git push --force-with-lease`) may be required