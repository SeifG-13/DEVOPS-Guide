# Stashing

## What is Stashing?

Stash temporarily saves uncommitted changes so you can work on something else.

```
Working Directory                  Stash Stack
┌─────────────────┐               ┌─────────────────┐
│  Uncommitted    │    stash      │   stash@{0}     │
│   Changes       │ ─────────────►│   stash@{1}     │
│                 │               │   stash@{2}     │
└─────────────────┘               └─────────────────┘
                       pop/apply
       ◄─────────────────────────────────────────────
```

## Creating Stashes

### Basic Stash

```bash
# Stash tracked modified files
git stash

# Stash with message
git stash save "Work in progress on feature"
git stash push -m "Work in progress on feature"

# Stash including untracked files
git stash -u
git stash --include-untracked

# Stash everything (including ignored)
git stash -a
git stash --all

# Stash specific files
git stash push -m "Partial stash" file1.txt file2.txt

# Stash with patch (interactive)
git stash -p
git stash --patch
```

### Keep Index (Staged Files)

```bash
# Stash but keep staged files staged
git stash --keep-index
git stash -k
```

## Viewing Stashes

```bash
# List all stashes
git stash list
# stash@{0}: WIP on main: abc123 Last commit message
# stash@{1}: On feature: def456 Another message

# Show stash contents (latest)
git stash show

# Show stash diff
git stash show -p

# Show specific stash
git stash show stash@{1}
git stash show -p stash@{1}
```

## Applying Stashes

### Apply and Keep Stash

```bash
# Apply latest stash (keep in stash list)
git stash apply

# Apply specific stash
git stash apply stash@{1}

# Apply and try to restore index
git stash apply --index
```

### Pop (Apply and Remove)

```bash
# Apply latest and remove from stash
git stash pop

# Pop specific stash
git stash pop stash@{1}
```

## Removing Stashes

```bash
# Drop latest stash
git stash drop

# Drop specific stash
git stash drop stash@{1}

# Clear all stashes
git stash clear
```

## Advanced Stashing

### Create Branch from Stash

```bash
# Create new branch and apply stash
git stash branch new-branch-name

# From specific stash
git stash branch new-branch-name stash@{1}
```

### Stash Untracked Only

```bash
# Stash only untracked files
git stash -u -- $(git ls-files --others --exclude-standard)
```

### Partial Stashing

```bash
# Interactive stash (choose hunks)
git stash -p

# Stash specific files only
git stash push -m "message" path/to/file1 path/to/file2
```

## Handling Conflicts

When applying stash causes conflicts:

```bash
# Apply stash (may conflict)
git stash pop
# CONFLICT in file.txt

# Resolve conflicts manually
# Edit file.txt

# Mark as resolved
git add file.txt

# The stash is NOT automatically dropped on conflict
# Drop manually after resolving
git stash drop
```

## Practical Scenarios

### Quick Context Switch

```bash
# Working on feature, need to fix urgent bug
git stash -m "Feature work in progress"

# Switch to main and fix bug
git checkout main
git checkout -b hotfix/urgent-fix
# ... fix bug ...
git commit -m "Fix urgent bug"
git checkout main
git merge hotfix/urgent-fix
git push

# Return to feature work
git checkout feature
git stash pop
```

### Save Work Before Pull

```bash
# Have local changes, need to pull
git stash
git pull
git stash pop
```

### Move Changes to New Branch

```bash
# Started work on wrong branch
git stash
git checkout -b correct-branch
git stash pop
git add .
git commit -m "Feature on correct branch"
```

### Test Clean Working Directory

```bash
# Stash changes
git stash -u

# Test with clean state
npm test

# Restore changes
git stash pop
```

### Review Stash Contents

```bash
# List stashes
git stash list

# See what's in latest stash
git stash show -p

# See specific stash
git stash show -p stash@{2}

# Compare stash with current
git diff stash@{0}
```

## Stash Best Practices

1. **Add descriptive messages**
   ```bash
   git stash push -m "WIP: user authentication logic"
   ```

2. **Don't let stashes pile up**
   - Review stash list regularly
   - Drop or apply old stashes

3. **Use branches for longer work**
   - Stash is for quick context switches
   - For longer work, use feature branches

4. **Include untracked files when needed**
   ```bash
   git stash -u
   ```

5. **Prefer `pop` over `apply`**
   - Keeps stash list clean
   - Use `apply` when you might need the stash again

## Stash Internals

Stashes are stored as commits:

```bash
# Stash reference
cat .git/refs/stash

# View stash as commit
git log --oneline stash

# Stash is actually two/three commits:
# - Index state
# - Working tree state
# - Untracked files (if -u used)
```

## Quick Reference

| Command | Purpose |
|---------|---------|
| `git stash` | Stash changes |
| `git stash -u` | Include untracked files |
| `git stash -m "msg"` | Stash with message |
| `git stash list` | List stashes |
| `git stash show` | Show stash summary |
| `git stash show -p` | Show stash diff |
| `git stash apply` | Apply stash (keep) |
| `git stash pop` | Apply and remove |
| `git stash drop` | Remove stash |
| `git stash clear` | Remove all stashes |
| `git stash branch name` | Create branch from stash |
| `git stash -p` | Interactive stash |

---

**Previous:** [07-undoing-changes.md](07-undoing-changes.md) | **Next:** [09-tagging-releases.md](09-tagging-releases.md)
