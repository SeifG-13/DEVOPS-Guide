# Advanced Git Commands

## Git Cherry-Pick

Apply specific commits to current branch.

```bash
# Cherry-pick single commit
git cherry-pick abc123

# Cherry-pick multiple commits
git cherry-pick abc123 def456 ghi789

# Cherry-pick range (exclusive start)
git cherry-pick abc123..def456

# Cherry-pick without committing
git cherry-pick -n abc123
git cherry-pick --no-commit abc123

# Cherry-pick and edit message
git cherry-pick -e abc123

# Continue after conflict
git cherry-pick --continue

# Abort cherry-pick
git cherry-pick --abort

# Keep original author
git cherry-pick -x abc123  # Adds reference in message
```

### Use Cases

```bash
# Backport fix to release branch
git checkout release-1.0
git cherry-pick abc123

# Pick specific feature commits
git cherry-pick feature-commit-1 feature-commit-2
```

## Git Bisect

Binary search to find commit that introduced a bug.

```bash
# Start bisect
git bisect start

# Mark current as bad
git bisect bad

# Mark known good commit
git bisect good abc123

# Git checks out middle commit
# Test if bug exists, then mark:
git bisect good  # or
git bisect bad

# Repeat until found
# Bisecting: X revisions left to test

# When done
git bisect reset

# Skip untestable commit
git bisect skip
```

### Automated Bisect

```bash
# Run script on each commit
git bisect start HEAD abc123
git bisect run npm test
git bisect reset
```

## Git Blame

Show who changed each line.

```bash
# Basic blame
git blame filename.txt

# Show specific lines
git blame -L 10,20 filename.txt

# Ignore whitespace
git blame -w filename.txt

# Show email
git blame -e filename.txt

# Show commit date
git blame filename.txt

# Ignore file renames
git blame -C filename.txt

# Follow content moves across files
git blame -C -C filename.txt
```

## Git Log Advanced

### Formatting

```bash
# One line per commit
git log --oneline

# Graph view
git log --graph --oneline --all

# Custom format
git log --pretty=format:"%h %an %ar - %s"

# Format placeholders:
# %H  - Full hash
# %h  - Short hash
# %an - Author name
# %ae - Author email
# %ar - Relative date
# %s  - Subject
# %b  - Body
```

### Filtering

```bash
# By author
git log --author="John"

# By date
git log --since="2024-01-01"
git log --until="2024-01-31"
git log --after="1 week ago"
git log --before="yesterday"

# By message
git log --grep="fix"
git log --grep="bug" --grep="fix" --all-match

# By file
git log -- path/to/file

# By content change
git log -S "function_name"  # Pickaxe
git log -G "regex_pattern"

# Merge commits only
git log --merges

# No merge commits
git log --no-merges

# First parent only
git log --first-parent
```

### Comparing

```bash
# Commits in B not in A
git log A..B

# Commits in either but not both
git log A...B

# With diff
git log -p

# Stats
git log --stat
git log --shortstat

# Name only
git log --name-only
git log --name-status
```

## Git Diff Advanced

```bash
# Word diff
git diff --word-diff

# Color words
git diff --color-words

# Stats only
git diff --stat

# Name only
git diff --name-only

# Name with status
git diff --name-status

# Ignore whitespace
git diff -w
git diff --ignore-all-space

# Ignore blank lines
git diff --ignore-blank-lines

# Between branches
git diff main..feature

# Between commits
git diff abc123 def456

# Specific file between commits
git diff abc123 def456 -- file.txt
```

## Git Submodules

Include other repositories inside your repository.

```bash
# Add submodule
git submodule add https://github.com/user/repo.git path/to/submodule

# Clone with submodules
git clone --recursive https://github.com/user/repo.git

# Initialize submodules (after clone)
git submodule init
git submodule update

# Or combined
git submodule update --init --recursive

# Update submodule to latest
cd path/to/submodule
git pull origin main
cd ..
git add path/to/submodule
git commit -m "Update submodule"

# Update all submodules
git submodule update --remote

# Remove submodule
git submodule deinit path/to/submodule
git rm path/to/submodule
rm -rf .git/modules/path/to/submodule
```

## Git Worktree

Multiple working directories from one repo.

```bash
# Add worktree
git worktree add ../feature-branch feature

# Add worktree with new branch
git worktree add -b new-branch ../new-branch main

# List worktrees
git worktree list

# Remove worktree
git worktree remove ../feature-branch

# Prune stale worktrees
git worktree prune
```

### Use Case

```bash
# Work on feature while testing main
git worktree add ../main-test main
cd ../main-test
npm test
cd ../repo
# Continue feature work
```

## Git Aliases

```bash
# Create aliases
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.last "log -1 HEAD"
git config --global alias.unstage "reset HEAD --"
git config --global alias.visual "!gitk"

# Complex alias
git config --global alias.recent "for-each-ref --sort=-committerdate refs/heads/ --format='%(committerdate:short) %(refname:short)'"

# Use aliases
git co main
git lg
git st
```

## Git Archive

Create archive of repository.

```bash
# Create zip
git archive --format=zip HEAD > archive.zip

# Create tar.gz
git archive --format=tar.gz HEAD > archive.tar.gz

# Specific prefix
git archive --prefix=project/ HEAD > archive.tar.gz

# Specific files
git archive HEAD path/to/file > archive.tar

# From specific commit/tag
git archive v1.0.0 > release.tar.gz
```

## Git Grep

Search repository content.

```bash
# Search in tracked files
git grep "pattern"

# Case insensitive
git grep -i "pattern"

# Show line numbers
git grep -n "pattern"

# Count matches
git grep -c "pattern"

# Show function context
git grep -p "pattern"

# Search specific file types
git grep "pattern" -- "*.js"

# Search in specific commit
git grep "pattern" abc123

# Search with context
git grep -A 3 -B 3 "pattern"
```

## Git Clean

```bash
# Dry run
git clean -n

# Remove untracked files
git clean -f

# Remove untracked directories
git clean -fd

# Remove ignored files too
git clean -fdx

# Interactive
git clean -i
```

## Quick Reference

| Command | Purpose |
|---------|---------|
| `git cherry-pick abc` | Apply specific commit |
| `git bisect start` | Start binary search |
| `git blame file` | Show line authors |
| `git log --oneline` | Compact log |
| `git log -S "text"` | Search for added text |
| `git diff --stat` | Diff statistics |
| `git submodule add url` | Add submodule |
| `git worktree add path branch` | Add worktree |
| `git archive HEAD` | Create archive |
| `git grep pattern` | Search content |

---

**Previous:** [10-git-workflows.md](10-git-workflows.md) | **Next:** [12-hooks.md](12-hooks.md)
