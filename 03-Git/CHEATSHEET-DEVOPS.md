# Git Cheat Sheet for DevOps Engineers

## Quick Reference Commands

### Configuration
```bash
git config --global user.name "Your Name"
git config --global user.email "email@example.com"
git config --global core.editor "vim"
git config --global init.defaultBranch main
git config --list                   # Show all config
```

### Repository Setup
```bash
git init                            # Initialize new repo
git clone url                       # Clone repository
git clone --depth 1 url             # Shallow clone (faster)
git remote add origin url           # Add remote
git remote -v                       # List remotes
git remote set-url origin new-url   # Change remote URL
```

### Basic Workflow
```bash
git status                          # Check status
git add file.txt                    # Stage file
git add .                           # Stage all changes
git add -p                          # Interactive staging
git commit -m "message"             # Commit with message
git commit -am "message"            # Add + commit tracked files
git push origin main                # Push to remote
git pull origin main                # Pull from remote
git fetch origin                    # Fetch without merge
```

### Branching
```bash
git branch                          # List local branches
git branch -a                       # List all branches
git branch feature                  # Create branch
git checkout feature                # Switch to branch
git checkout -b feature             # Create and switch
git switch feature                  # Modern switch command
git switch -c feature               # Create and switch (modern)
git branch -d feature               # Delete branch (safe)
git branch -D feature               # Force delete branch
git push origin --delete feature    # Delete remote branch
```

### Merging & Rebasing
```bash
git merge feature                   # Merge branch
git merge --no-ff feature           # Merge with merge commit
git rebase main                     # Rebase onto main
git rebase -i HEAD~3                # Interactive rebase
git cherry-pick commit-hash         # Apply specific commit
git merge --abort                   # Abort merge
git rebase --abort                  # Abort rebase
```

### Stashing
```bash
git stash                           # Stash changes
git stash save "message"            # Stash with message
git stash list                      # List stashes
git stash pop                       # Apply and remove stash
git stash apply                     # Apply without removing
git stash drop                      # Delete stash
git stash clear                     # Delete all stashes
```

### History & Logs
```bash
git log                             # Commit history
git log --oneline                   # Compact log
git log --graph --oneline --all     # Visual branch graph
git log -p file.txt                 # File change history
git log --author="name"             # Filter by author
git log --since="2 weeks ago"       # Filter by date
git show commit-hash                # Show commit details
git blame file.txt                  # Who changed what
```

### Undoing Changes
```bash
git checkout -- file.txt            # Discard file changes
git restore file.txt                # Modern discard (Git 2.23+)
git reset HEAD file.txt             # Unstage file
git restore --staged file.txt       # Modern unstage
git reset --soft HEAD~1             # Undo commit, keep changes staged
git reset --mixed HEAD~1            # Undo commit, keep changes unstaged
git reset --hard HEAD~1             # Undo commit, discard changes
git revert commit-hash              # Create inverse commit
```

### Tags
```bash
git tag                             # List tags
git tag v1.0.0                      # Create lightweight tag
git tag -a v1.0.0 -m "Version 1.0"  # Annotated tag
git push origin v1.0.0              # Push specific tag
git push origin --tags              # Push all tags
git tag -d v1.0.0                   # Delete local tag
git push origin --delete v1.0.0     # Delete remote tag
```

### Comparing
```bash
git diff                            # Unstaged changes
git diff --staged                   # Staged changes
git diff main..feature              # Compare branches
git diff HEAD~3..HEAD               # Last 3 commits
git diff --stat                     # Summary of changes
```

---

## Git Workflow Strategies

### GitFlow
```
main (production)
  └── develop (integration)
        ├── feature/xxx (new features)
        ├── release/x.x (release prep)
        └── hotfix/xxx (urgent fixes)
```

### GitHub Flow
```
main (always deployable)
  └── feature-branch
        └── Pull Request → Code Review → Merge → Deploy
```

### Trunk-Based Development
```
main (single source of truth)
  └── short-lived feature branches (< 1 day)
        └── Merge frequently with feature flags
```

---

## Interview Q&A

### Q1: What is the difference between merge and rebase?
**A:**
- **Merge**: Creates a merge commit, preserves complete history, non-destructive
- **Rebase**: Rewrites history by moving commits, creates linear history, destructive (don't rebase shared branches)

### Q2: What is the difference between git pull and git fetch?
**A:**
- **fetch**: Downloads changes without merging, safe operation
- **pull**: Fetches + merges (or rebases with --rebase flag)

### Q3: How do you resolve merge conflicts?
**A:**
1. `git status` - identify conflicted files
2. Open files, look for `<<<<<<<`, `=======`, `>>>>>>>`
3. Edit to resolve conflicts
4. `git add <files>` - mark as resolved
5. `git commit` - complete the merge

### Q4: What is git stash and when do you use it?
**A:** Temporarily saves uncommitted changes. Use when:
- Switching branches with uncommitted work
- Pulling when you have local changes
- Setting aside work to handle urgent issues

### Q5: How do you revert a pushed commit?
**A:**
```bash
# Safe way - creates new commit
git revert commit-hash
git push

# If you must rewrite history (DANGEROUS for shared branches)
git reset --hard HEAD~1
git push --force
```

### Q6: What is the difference between reset and revert?
**A:**
- **reset**: Moves HEAD pointer, rewrites history, dangerous for shared branches
- **revert**: Creates new commit undoing changes, safe for shared branches

### Q7: How do you squash commits?
**A:**
```bash
git rebase -i HEAD~3
# Change 'pick' to 'squash' for commits to combine
# Edit commit message
git push --force  # Only if already pushed
```

### Q8: What are Git hooks?
**A:** Scripts that run automatically on Git events:
- **pre-commit**: Before commit (linting, tests)
- **pre-push**: Before push (tests, security checks)
- **post-receive**: After receiving push (deploy)

Located in `.git/hooks/` or managed by tools like Husky.

### Q9: How do you handle large files in Git?
**A:**
- **Git LFS**: Large File Storage extension
- **.gitignore**: Exclude unnecessary files
- **BFG Repo Cleaner**: Remove large files from history

### Q10: What is a detached HEAD state?
**A:** When HEAD points to a commit instead of a branch. Happens when:
- Checking out a specific commit
- Checking out a tag

Fix: Create a branch from current state: `git checkout -b new-branch`

---

## CI/CD Integration

### GitHub Actions Trigger
```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

### GitLab CI Trigger
```yaml
rules:
  - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  - if: '$CI_COMMIT_BRANCH == "main"'
```

### Pre-commit Hooks (DevOps)
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: check-yaml
      - id: check-added-large-files
```

---

## Useful Aliases
```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --all"
git config --global alias.last "log -1 HEAD"
git config --global alias.unstage "reset HEAD --"
```

---

## Best Practices

1. **Write meaningful commit messages** - Describe what and why
2. **Commit often, push regularly** - Small, atomic commits
3. **Never force push shared branches** - Coordinate with team
4. **Use branches for features** - Keep main stable
5. **Review before merging** - Pull requests with code review
6. **Use .gitignore** - Exclude build artifacts, secrets, dependencies
7. **Tag releases** - Semantic versioning (v1.0.0)
8. **Protect main branch** - Require reviews, status checks
