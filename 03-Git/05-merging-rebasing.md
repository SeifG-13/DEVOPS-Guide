# Merging & Rebasing

## Git Merge

Merge combines branches by creating a new commit.

### Types of Merges

#### Fast-Forward Merge

When target branch has no new commits.

```
Before:
main:    A───B
              \
feature:       C───D

After (fast-forward):
main:    A───B───C───D
                      │
                   feature
```

```bash
git checkout main
git merge feature
# Fast-forward
```

#### Three-Way Merge

When both branches have new commits.

```
Before:
main:    A───B───E───F
              \
feature:       C───D

After (merge commit):
main:    A───B───E───F───M
              \         /
feature:       C───D───┘
```

```bash
git checkout main
git merge feature
# Creates merge commit M
```

### Merge Commands

```bash
# Merge branch into current
git merge feature

# Merge with commit message
git merge feature -m "Merge feature branch"

# No fast-forward (always create merge commit)
git merge --no-ff feature

# Fast-forward only (fail if not possible)
git merge --ff-only feature

# Squash (combine all commits into one)
git merge --squash feature
git commit -m "Squashed feature"

# Abort merge
git merge --abort

# Continue after resolving conflicts
git merge --continue
```

## Git Rebase

Rebase moves commits to a new base, rewriting history.

```
Before:
main:    A───B───E───F
              \
feature:       C───D

After rebase:
main:    A───B───E───F
                      \
feature:               C'───D'
```

```bash
# Rebase current branch onto main
git checkout feature
git rebase main

# Or specify both
git rebase main feature
```

### Interactive Rebase

Edit, squash, reorder, or drop commits.

```bash
# Rebase last 3 commits
git rebase -i HEAD~3

# Rebase onto branch
git rebase -i main
```

Interactive options:

```
pick abc123 First commit      # keep commit
reword def456 Second commit   # edit message
edit ghi789 Third commit      # stop to amend
squash jkl012 Fourth commit   # combine with previous
fixup mno345 Fifth commit     # squash, discard message
drop pqr678 Sixth commit      # remove commit
```

### Rebase Commands

```bash
# Basic rebase
git rebase main

# Interactive rebase
git rebase -i main
git rebase -i HEAD~5

# Continue after resolving conflicts
git rebase --continue

# Skip current commit
git rebase --skip

# Abort rebase
git rebase --abort

# Preserve merge commits
git rebase --preserve-merges main
git rebase --rebase-merges main   # newer version

# Rebase onto specific commit
git rebase --onto newbase oldbase branch
```

## Merge vs Rebase

| Aspect | Merge | Rebase |
|--------|-------|--------|
| History | Preserves exact history | Creates linear history |
| Commits | Adds merge commit | No extra commits |
| Conflicts | Resolve once | Resolve per commit |
| Safety | Never rewrites history | Rewrites history |
| Collaboration | Safe for shared branches | Dangerous for shared |
| Use when | Preserving context matters | Clean history wanted |

### When to Use Merge

- Merging to main/shared branches
- Preserving complete history
- Pull requests
- Team collaboration

### When to Use Rebase

- Updating feature branch with main
- Cleaning up local commits before PR
- Keeping linear history
- Personal/local branches only

## Resolving Conflicts

### Conflict Markers

```
<<<<<<< HEAD
Your changes (current branch)
=======
Their changes (incoming branch)
>>>>>>> feature
```

### Resolving Steps

```bash
# 1. Git shows conflict
git merge feature
# CONFLICT (content): Merge conflict in file.txt

# 2. Check status
git status

# 3. Open conflicted file(s) and resolve
# Edit file to remove markers and keep desired content

# 4. Stage resolved files
git add file.txt

# 5. Complete merge
git commit
# Or for rebase:
git rebase --continue
```

### Conflict Resolution Tools

```bash
# Use merge tool
git mergetool

# Configure merge tool
git config --global merge.tool vimdiff
git config --global merge.tool vscode

# VS Code as merge tool
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# Accept current or incoming
git checkout --ours file.txt    # Keep current branch
git checkout --theirs file.txt  # Keep incoming branch
```

## Squash Commits

Combine multiple commits into one.

### Using Interactive Rebase

```bash
# Squash last 3 commits
git rebase -i HEAD~3

# In editor, change 'pick' to 'squash' (or 's')
pick abc123 First commit
squash def456 Second commit
squash ghi789 Third commit

# Save and edit combined commit message
```

### Using Merge --squash

```bash
git checkout main
git merge --squash feature
git commit -m "Feature: combined changes"
```

## Cherry-Pick

Apply specific commit(s) to current branch.

```bash
# Cherry-pick single commit
git cherry-pick abc123

# Cherry-pick multiple commits
git cherry-pick abc123 def456

# Cherry-pick range
git cherry-pick abc123..def456

# Cherry-pick without committing
git cherry-pick --no-commit abc123

# Continue after conflict
git cherry-pick --continue

# Abort cherry-pick
git cherry-pick --abort
```

## Practical Examples

### Update Feature Branch with Main

```bash
# Option 1: Merge (safer)
git checkout feature
git merge main

# Option 2: Rebase (cleaner)
git checkout feature
git rebase main
git push --force-with-lease  # If already pushed
```

### Clean Up Commits Before PR

```bash
# Squash commits
git rebase -i main

# Change to squash/fixup
pick abc123 Add feature
squash def456 Fix typo
squash ghi789 Add tests

# Result: one clean commit
```

### Undo a Merge

```bash
# If not pushed yet
git reset --hard HEAD~1

# If already pushed (creates revert commit)
git revert -m 1 <merge-commit>
```

### Resolve Complex Conflicts

```bash
# Start merge
git merge feature

# If stuck, check file versions
git show :1:file.txt  # Common ancestor
git show :2:file.txt  # Ours (current)
git show :3:file.txt  # Theirs (incoming)

# Use mergetool
git mergetool

# Complete
git add .
git commit
```

## Quick Reference

### Merge Commands

| Command | Purpose |
|---------|---------|
| `git merge branch` | Merge branch |
| `git merge --no-ff` | Force merge commit |
| `git merge --squash` | Squash merge |
| `git merge --abort` | Abort merge |

### Rebase Commands

| Command | Purpose |
|---------|---------|
| `git rebase main` | Rebase onto main |
| `git rebase -i HEAD~3` | Interactive rebase |
| `git rebase --continue` | Continue rebase |
| `git rebase --abort` | Abort rebase |

### Conflict Resolution

| Command | Purpose |
|---------|---------|
| `git status` | See conflicted files |
| `git mergetool` | Open merge tool |
| `git checkout --ours file` | Keep current |
| `git checkout --theirs file` | Keep incoming |

---

**Previous:** [04-branching.md](04-branching.md) | **Next:** [06-remote-repositories.md](06-remote-repositories.md)
