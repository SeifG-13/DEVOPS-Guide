# Basic Git Commands

## Creating Repositories

### git init

Initialize a new Git repository.

```bash
# Create new repository
mkdir myproject
cd myproject
git init

# Initialize with main branch
git init -b main

# Initialize bare repository (for servers)
git init --bare myproject.git
```

### git clone

Copy an existing repository.

```bash
# Clone via HTTPS
git clone https://github.com/user/repo.git

# Clone via SSH
git clone git@github.com:user/repo.git

# Clone to specific directory
git clone https://github.com/user/repo.git my-folder

# Clone specific branch
git clone -b develop https://github.com/user/repo.git

# Shallow clone (latest commit only)
git clone --depth 1 https://github.com/user/repo.git

# Clone with submodules
git clone --recursive https://github.com/user/repo.git
```

## Tracking Changes

### git status

Show the working tree status.

```bash
# Full status
git status

# Short format
git status -s
git status --short

# Output:
# ?? untracked.txt      (untracked)
# A  staged.txt         (staged, new)
# M  modified.txt       (staged, modified)
#  M modified2.txt      (not staged, modified)
# MM both.txt           (staged and unstaged changes)
# D  deleted.txt        (staged for deletion)
```

### git add

Add files to staging area.

```bash
# Add specific file
git add filename.txt

# Add multiple files
git add file1.txt file2.txt

# Add all files in directory
git add .

# Add all tracked files (modified/deleted)
git add -u

# Add all files (including untracked)
git add -A
git add --all

# Add with pattern
git add *.js
git add src/

# Interactive staging
git add -i

# Stage parts of file (hunks)
git add -p filename.txt
```

### git commit

Record changes to repository.

```bash
# Commit with message
git commit -m "Add new feature"

# Commit with multi-line message
git commit -m "Title" -m "Description paragraph"

# Open editor for message
git commit

# Add all tracked files and commit
git commit -a -m "Message"
git commit -am "Message"

# Amend last commit
git commit --amend

# Amend without changing message
git commit --amend --no-edit

# Signed commit
git commit -S -m "Signed commit"

# Empty commit (useful for triggering CI)
git commit --allow-empty -m "Trigger build"
```

### git diff

Show changes between commits, branches, files.

```bash
# Unstaged changes (working dir vs staging)
git diff

# Staged changes (staging vs last commit)
git diff --staged
git diff --cached

# All changes (working dir vs last commit)
git diff HEAD

# Between commits
git diff abc123 def456
git diff HEAD~3 HEAD

# Between branches
git diff main develop

# Specific file
git diff filename.txt
git diff HEAD~1 filename.txt

# Show only file names
git diff --name-only

# Show statistics
git diff --stat

# Word diff
git diff --word-diff
```

## Viewing History

### git log

Show commit history.

```bash
# Full log
git log

# One line per commit
git log --oneline

# With graph
git log --graph

# Decorated (show branches/tags)
git log --decorate

# Combined
git log --oneline --graph --decorate --all

# Last N commits
git log -5

# By author
git log --author="John"

# By date
git log --since="2024-01-01"
git log --until="2024-01-31"
git log --after="1 week ago"

# By message
git log --grep="bug fix"

# By file
git log -- filename.txt

# Show changes in each commit
git log -p

# Show stats
git log --stat

# Custom format
git log --pretty=format:"%h - %an, %ar : %s"

# Compact format
git log --pretty=short
git log --pretty=full
```

### git show

Show commit details.

```bash
# Show last commit
git show

# Show specific commit
git show abc123

# Show specific file at commit
git show abc123:filename.txt

# Show tag
git show v1.0.0

# Show only stats
git show --stat
```

### git blame

Show who changed each line.

```bash
# Show file with annotations
git blame filename.txt

# Show specific lines
git blame -L 10,20 filename.txt

# Show email
git blame -e filename.txt

# Ignore whitespace changes
git blame -w filename.txt
```

## Managing Files

### git rm

Remove files from working tree and index.

```bash
# Remove file
git rm filename.txt

# Remove from index only (keep file)
git rm --cached filename.txt

# Remove directory
git rm -r directory/

# Force remove
git rm -f filename.txt

# Dry run
git rm -n filename.txt
```

### git mv

Move or rename files.

```bash
# Rename file
git mv oldname.txt newname.txt

# Move file
git mv file.txt directory/

# Move and rename
git mv old/path/file.txt new/path/newname.txt
```

## Common Workflows

### Starting New Project

```bash
# Create directory
mkdir myproject && cd myproject

# Initialize Git
git init

# Create initial files
echo "# My Project" > README.md

# Stage and commit
git add README.md
git commit -m "Initial commit"

# Add remote
git remote add origin git@github.com:user/myproject.git

# Push to remote
git push -u origin main
```

### Daily Workflow

```bash
# Start of day - get latest
git pull

# Create feature branch
git checkout -b feature/new-feature

# Make changes
# ... edit files ...

# Check status
git status

# View changes
git diff

# Stage changes
git add .

# Commit
git commit -m "Add new feature"

# Push to remote
git push -u origin feature/new-feature

# Create PR on GitHub
```

### Quick Fixes

```bash
# Undo last commit (keep changes)
git reset --soft HEAD~1

# Discard all local changes
git checkout -- .
git restore .

# Discard changes to specific file
git checkout -- filename.txt
git restore filename.txt

# Remove untracked files
git clean -fd
```

## Quick Reference

| Command | Purpose |
|---------|---------|
| `git init` | Create new repository |
| `git clone url` | Copy repository |
| `git status` | Show status |
| `git add file` | Stage file |
| `git add .` | Stage all files |
| `git commit -m "msg"` | Commit changes |
| `git diff` | Show unstaged changes |
| `git diff --staged` | Show staged changes |
| `git log` | Show history |
| `git log --oneline` | Compact history |
| `git show` | Show commit details |
| `git rm file` | Remove file |
| `git mv old new` | Move/rename file |

---

**Previous:** [02-installation-config.md](02-installation-config.md) | **Next:** [04-branching.md](04-branching.md)
