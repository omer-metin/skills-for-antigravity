---
name: git-workflow
description: "Guides branching strategies, commit hygiene, merge and rebase decisions, conflict resolution, and history management for Git-based projects. Enforces trunk-based development with short-lived branches, conventional commits, and safe force-push practices. Use when git workflow, branching strategy, git merge, git rebase, merge conflict, git commit, trunk-based, gitflow, git history, git revert, git reset, version-control, branching, commits, collaboration mentioned."
---

# Git Workflow

## Principles

- Commit early and often — small commits are easier to review, bisect, and revert.
- Write commit messages for someone who has no context — answer *what* changed and *why*.
- Never force push to shared branches; use `--force-with-lease` on personal branches only.
- Prefer rebase for local, unpushed work; prefer merge for shared branches.
- Keep branches short-lived — merge or delete within days, not weeks. Branches older than three days are a process smell.
- The main branch should always be deployable.
- Use `git reflog` before panicking — almost nothing in Git is truly lost.

## Workflow

### 1. Starting New Work

Create a branch from the latest upstream state using a consistent naming convention:

```bash
# Fetch latest and create a feature branch
git fetch origin
git switch -c feature/AUTH-123-oauth-login origin/main

# Naming convention: <type>/<ticket>-<short-description>
# Types: feature/, fix/, hotfix/, chore/, refactor/, experiment/
```

### 2. Making Atomic Commits

Each commit should do one logical thing, leave the codebase in a working state, and be independently revertable.

```bash
# Stage specific changes (not everything at once)
git add -p                          # Interactive hunk-by-hunk staging

# Write a conventional commit message
git commit -m "feat(auth): add OAuth2 login with Google"

# Conventional commit format: <type>(<scope>): <description>
# Types: feat, fix, docs, style, refactor, perf, test, chore
# Breaking changes: add ! after type, e.g. feat!: change API response format
```

Avoid generic messages like "fix", "update", or "wip". Each message should answer: *what does this commit do and why?*

### 3. Keeping the Branch Current

```bash
# For unpushed local work — rebase onto latest main
git fetch origin
git rebase origin/main

# For already-pushed branches — merge instead to avoid rewriting shared history
git merge origin/main
```

**Golden rule:** never rebase commits that already exist on the remote. Rebasing pushed commits creates divergent history and forces a force-push, which risks overwriting teammates' work.

### 4. Cleaning Up Before a Pull Request

Squash fixup commits and typo fixes so the PR tells a clear story:

```bash
# Interactive rebase to clean last 5 commits
git rebase -i HEAD~5

# In the editor, mark typo/fixup commits with 'fixup' or 'squash'
# pick abc1234 Add user model
# fixup def5678 Fix typo
# pick ghi9012 Add user validation
# pick mno7890 Add user tests

# If conflicts arise during rebase:
git rebase --continue   # after resolving
git rebase --abort      # to bail out entirely
```

### 5. Opening a Pull Request

```bash
# Push the branch and set upstream tracking
git push -u origin feature/AUTH-123-oauth-login

# Open a pull request (GitHub CLI)
gh pr create --base main --title "feat(auth): add OAuth2 login" \
  --body "Adds Google OAuth2 flow with session management. Closes #123."
```

### 6. Resolving Merge Conflicts

Conflicts are normal — do not panic or run destructive commands mid-merge.

```bash
# See which files conflict
git status

# In each conflicted file, look for markers:
# <<<<<<< HEAD
# (your changes)
# =======
# (their changes)
# >>>>>>> branch-name

# Edit the file: remove markers, keep the correct code, save.
# Then mark resolved and complete the merge:
git add <resolved-file>
git commit

# To abort and start over:
git merge --abort
```

### 7. Safe Recovery

```bash
# View the reflog — records every change to HEAD
git reflog

# Recover from an accidental hard reset
git reset --hard <reflog-hash>

# Recover a deleted branch
git checkout -b recovered-branch <reflog-hash>

# Stash uncommitted work before risky operations
git stash push -m "WIP: auth validation"
git stash pop   # restore later
```

## Context Switching with Stash

When switching branches with uncommitted work, stash rather than making throwaway commits:

```bash
git stash push -m "WIP: auth validation"  # save current state
git stash -u                               # include untracked files
git stash list                             # see all stashes
git stash apply stash@{1}                  # apply a specific stash
git stash show -p stash@{0}               # inspect stash contents
```

## Common Pitfalls

| Pitfall | Risk | Mitigation |
|---------|------|------------|
| Working directly on main | No review, no CI, no rollback point | Always branch; open a PR |
| `git push --force` on shared branches | Rewrites history others depend on | Use `--force-with-lease` on personal branches only |
| Giant commits (50+ files) | Impossible to review, bisect, or revert | Commit after each logical unit of work |
| Long-lived feature branches (>3 days) | Merge conflicts compound; integration risk grows | Break into smaller incremental changes; use feature flags |
| Adding to `.gitignore` after tracking | Files remain tracked despite ignore rules | Run `git rm --cached <file>` then commit the `.gitignore` change |
| Amending already-pushed commits | Requires force push; risks shared history | Make a new commit instead; squash later in PR |

## Reference System Usage

Responses should be grounded in the provided reference files, treating them as the source of truth for this domain:

* **For creation tasks:** consult **`references/patterns.md`** — it defines how branching, commits, rebasing, and stashing should be done, with concrete examples.
* **For diagnosis:** consult **`references/sharp_edges.md`** — it catalogs critical failures (force-push disasters, reset data loss, detached HEAD, rebase confusion) with severity ratings, symptoms, and solutions.
* **For review and validation:** consult **`references/validations.md`** — it contains regex-based rules for detecting unsafe commands, weak commit messages, sensitive file exposure, and conflict markers.

If a user's request conflicts with guidance in these reference files, correct the approach using the information provided there.
