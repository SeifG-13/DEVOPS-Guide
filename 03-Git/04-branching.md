# Branching

## What is a Branch?

A branch is an independent line of development. It's a lightweight pointer to a commit.

```
                    feature
                       │
                       ▼
              ┌───┐   ┌───┐
              │ C │───│ D │
              └───┘   └───┘
             ╱
┌───┐   ┌───┐
│ A │───│ B │───────────────── main
└───┘   └───┘                    │
                                 ▼
                              ┌───┐
                              │ E │
                              └───┘
                                 ▲
                                 │
                               HEAD
```

## HEAD

HEAD is a pointer to the current branch (or commit in detached state).

```bash
# View HEAD
cat .git/HEAD
# ref: refs/heads/main

# See where HEAD points
git log -1 HEAD
```

## Listing Branches

```bash
# List local branches
git branch

# List all branches (local + remote)
git branch -a

# List remote branches only
git branch -r

# List with last commit
git branch -v

# List merged branches
git branch --merged

# List unmerged branches
git branch --no-merged

# List branches containing commit
git branch --contains abc123
```

## Creating Branches

```bash
# Create branch
git branch feature-name

# Create and switch to branch
git checkout -b feature-name
git switch -c feature-name

# Create branch from specific commit
git branch feature-name abc123

# Create branch from another branch
git branch feature-name origin/develop

# Create tracking branch
git branch --track feature origin/feature
```

## Switching Branches

```bash
# Switch to branch (old way)
git checkout branch-name

# Switch to branch (new way, Git 2.23+)
git switch branch-name

# Switch to previous branch
git checkout -
git switch -

# Create and switch
git checkout -b new-branch
git switch -c new-branch

# Switch to remote branch
git checkout origin/feature
git switch --detach origin/feature

# Force switch (discard changes)
git checkout -f branch-name
git switch -f branch-name
```

## Renaming Branches

```bash
# Rename current branch
git branch -m new-name

# Rename specific branch
git branch -m old-name new-name

# Rename and push
git branch -m old-name new-name
git push origin -u new-name
git push origin --delete old-name
```

## Deleting Branches

```bash
# Delete local branch (merged)
git branch -d branch-name

# Force delete (unmerged)
git branch -D branch-name

# Delete remote branch
git push origin --delete branch-name
git push origin :branch-name

# Delete local tracking reference
git branch -dr origin/branch-name

# Clean up deleted remote branches
git fetch --prune
git remote prune origin
```

## Branch Operations

### Viewing Branch Differences

```bash
# Commits in feature not in main
git log main..feature

# Commits in main not in feature
git log feature..main

# Commits in either but not both
git log main...feature

# Files changed between branches
git diff main..feature --name-only

# Compare branches
git diff main feature
```

### Setting Upstream

```bash
# Set upstream when pushing
git push -u origin feature
git push --set-upstream origin feature

# Set upstream for existing branch
git branch -u origin/feature
git branch --set-upstream-to=origin/feature

# See upstream
git branch -vv

# Unset upstream
git branch --unset-upstream
```

## Branching Strategies

### Common Branch Types

| Branch | Purpose | Example |
|--------|---------|---------|
| main/master | Production code | `main` |
| develop | Integration branch | `develop` |
| feature | New features | `feature/login` |
| bugfix | Bug fixes | `bugfix/header-crash` |
| hotfix | Urgent production fixes | `hotfix/security-patch` |
| release | Release preparation | `release/v1.2.0` |

### Naming Conventions

```bash
# Feature branches
feature/user-authentication
feature/add-payment-gateway
feature/JIRA-123-new-dashboard

# Bug fixes
bugfix/fix-login-error
bugfix/JIRA-456-null-pointer

# Hotfixes
hotfix/security-vulnerability
hotfix/critical-crash

# Releases
release/v1.0.0
release/2024-01-sprint
```

## Practical Examples

### Feature Branch Workflow

```bash
# Start from main
git checkout main
git pull origin main

# Create feature branch
git checkout -b feature/new-feature

# Work on feature
# ... make changes ...
git add .
git commit -m "Add new feature"

# Keep up with main
git fetch origin
git rebase origin/main

# Push feature branch
git push -u origin feature/new-feature

# After PR is merged, cleanup
git checkout main
git pull origin main
git branch -d feature/new-feature
```

### Hotfix Workflow

```bash
# Create hotfix from main
git checkout main
git pull origin main
git checkout -b hotfix/critical-fix

# Fix the issue
git add .
git commit -m "Fix critical bug"

# Push and create PR
git push -u origin hotfix/critical-fix

# After merge, tag the release
git checkout main
git pull origin main
git tag -a v1.0.1 -m "Hotfix release"
git push origin v1.0.1
```

## Detached HEAD

When HEAD points directly to a commit instead of a branch.

```bash
# Enter detached HEAD state
git checkout abc123
git checkout v1.0.0

# Warning message appears
# You are in 'detached HEAD' state...

# Create branch from detached HEAD
git checkout -b new-branch

# Return to branch
git checkout main
```

## Quick Reference

| Command | Purpose |
|---------|---------|
| `git branch` | List branches |
| `git branch name` | Create branch |
| `git checkout branch` | Switch branch |
| `git switch branch` | Switch branch (new) |
| `git checkout -b name` | Create and switch |
| `git switch -c name` | Create and switch (new) |
| `git branch -d name` | Delete branch |
| `git branch -D name` | Force delete branch |
| `git branch -m new` | Rename current branch |
| `git push origin --delete name` | Delete remote branch |
| `git branch -a` | List all branches |
| `git branch -vv` | Show upstream info |

---

**Previous:** [03-basic-commands.md](03-basic-commands.md) | **Next:** [05-merging-rebasing.md](05-merging-rebasing.md)
