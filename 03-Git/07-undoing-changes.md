# Undoing Changes

## Overview of Undo Options

```
┌─────────────────────────────────────────────────────────────────┐
│                    Git Undo Commands                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Working Dir    ──►   Staging    ──►   Repository               │
│                                                                  │
│   restore            restore            reset                    │
│   checkout           reset HEAD         revert                   │
│   clean                                 commit --amend           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Discard Working Directory Changes

### git restore (Git 2.23+)

```bash
# Discard changes to specific file
git restore filename.txt

# Discard all changes
git restore .

# Restore file from specific commit
git restore --source=HEAD~1 filename.txt

# Restore from branch
git restore --source=main filename.txt
```

### git checkout (Legacy)

```bash
# Discard changes to specific file
git checkout -- filename.txt

# Discard all changes
git checkout -- .
git checkout .

# Restore from specific commit
git checkout abc123 -- filename.txt
```

## Unstage Files

### git restore --staged

```bash
# Unstage specific file
git restore --staged filename.txt

# Unstage all files
git restore --staged .
```

### git reset HEAD

```bash
# Unstage specific file
git reset HEAD filename.txt

# Unstage all
git reset HEAD
git reset
```

## Remove Untracked Files

### git clean

```bash
# Dry run (show what would be deleted)
git clean -n

# Remove untracked files
git clean -f

# Remove untracked files and directories
git clean -fd

# Remove ignored files too
git clean -fdx

# Interactive mode
git clean -i
```

## Modify Last Commit

### git commit --amend

```bash
# Change last commit message
git commit --amend -m "New message"

# Add files to last commit
git add forgotten-file.txt
git commit --amend --no-edit

# Change author
git commit --amend --author="Name <email@example.com>"

# Reset author to current config
git commit --amend --reset-author

# Amend without changing message
git commit --amend --no-edit
```

**Warning:** Don't amend commits that are already pushed!

## Git Reset

Reset moves HEAD and optionally modifies staging and working directory.

### Reset Modes

| Mode | HEAD | Staging | Working Dir |
|------|------|---------|-------------|
| --soft | ✓ Moves | ✗ Unchanged | ✗ Unchanged |
| --mixed (default) | ✓ Moves | ✓ Reset | ✗ Unchanged |
| --hard | ✓ Moves | ✓ Reset | ✓ Reset |

### Soft Reset

Undo commit, keep changes staged.

```bash
# Undo last commit, keep staged
git reset --soft HEAD~1

# Undo last 3 commits
git reset --soft HEAD~3

# Reset to specific commit
git reset --soft abc123
```

### Mixed Reset (Default)

Undo commit, unstage changes, keep in working directory.

```bash
# Undo last commit, unstage changes
git reset HEAD~1
git reset --mixed HEAD~1

# Unstage all files
git reset
```

### Hard Reset

Undo commit, discard all changes (dangerous!).

```bash
# Undo last commit, discard everything
git reset --hard HEAD~1

# Reset to remote state
git reset --hard origin/main

# Reset to specific commit
git reset --hard abc123

# Discard all local changes
git reset --hard HEAD
```

### Reset Specific File

```bash
# Reset file to last commit (unstage and restore)
git reset HEAD filename.txt

# This is equivalent to:
git restore --staged filename.txt
```

## Git Revert

Create a new commit that undoes previous changes. Safe for shared branches.

```bash
# Revert last commit
git revert HEAD

# Revert specific commit
git revert abc123

# Revert without committing
git revert --no-commit abc123
git revert -n abc123

# Revert multiple commits
git revert abc123 def456

# Revert range (exclusive start)
git revert abc123..def456

# Revert merge commit (specify parent)
git revert -m 1 <merge-commit>
```

## Git Reflog

Reflog records all HEAD movements. Your safety net!

```bash
# View reflog
git reflog

# Output:
# abc123 HEAD@{0}: commit: Add feature
# def456 HEAD@{1}: checkout: moving from main to feature
# ghi789 HEAD@{2}: commit: Previous commit

# View reflog for specific branch
git reflog show main

# Recover lost commit
git checkout HEAD@{2}
git branch recovered-branch HEAD@{2}

# Recover after hard reset
git reflog
# Find the commit before reset
git reset --hard HEAD@{1}
```

## Reset vs Revert

| Aspect | Reset | Revert |
|--------|-------|--------|
| History | Rewrites history | Adds new commit |
| Shared branches | Dangerous | Safe |
| Pushed commits | Don't use | Use this |
| Recovery | Use reflog | Just delete revert commit |

## Practical Scenarios

### Undo Last Commit (Not Pushed)

```bash
# Keep changes staged
git reset --soft HEAD~1

# Keep changes unstaged
git reset HEAD~1

# Discard changes completely
git reset --hard HEAD~1
```

### Undo Last Commit (Already Pushed)

```bash
# Create revert commit
git revert HEAD
git push
```

### Discard All Local Changes

```bash
# Discard tracked file changes
git checkout -- .
# or
git restore .

# Remove untracked files
git clean -fd

# Complete reset to remote
git fetch origin
git reset --hard origin/main
git clean -fd
```

### Recover Deleted Branch

```bash
# Find the commit
git reflog

# Recreate branch
git branch recovered-branch abc123
```

### Unstage Accidentally Staged File

```bash
git restore --staged filename.txt
# or
git reset HEAD filename.txt
```

### Fix Wrong Branch Commit

```bash
# Committed to main instead of feature
# 1. Create branch with the commit
git branch feature

# 2. Remove commit from main
git reset --hard HEAD~1

# 3. Switch to feature
git checkout feature
```

### Recover from Bad Merge

```bash
# If not pushed
git reset --hard HEAD~1

# If pushed
git revert -m 1 <merge-commit>
git push
```

## Quick Reference

| Scenario | Command |
|----------|---------|
| Discard file changes | `git restore file` |
| Discard all changes | `git restore .` |
| Unstage file | `git restore --staged file` |
| Unstage all | `git reset` |
| Remove untracked | `git clean -fd` |
| Amend last commit | `git commit --amend` |
| Undo commit (keep staged) | `git reset --soft HEAD~1` |
| Undo commit (keep unstaged) | `git reset HEAD~1` |
| Undo commit (discard all) | `git reset --hard HEAD~1` |
| Revert pushed commit | `git revert <commit>` |
| View reflog | `git reflog` |
| Recover lost commit | `git checkout HEAD@{n}` |

---

**Previous:** [06-remote-repositories.md](06-remote-repositories.md) | **Next:** [08-stashing.md](08-stashing.md)
