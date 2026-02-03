# Remote Repositories

## Understanding Remotes

A remote is a version of your repository hosted elsewhere (GitHub, GitLab, etc.).

```
┌─────────────────┐         ┌─────────────────┐
│  Local Repo     │◄───────►│  Remote Repo    │
│                 │  push   │  (origin)       │
│  ┌───────────┐  │  pull   │  ┌───────────┐  │
│  │   main    │  │  fetch  │  │   main    │  │
│  └───────────┘  │         │  └───────────┘  │
│  ┌───────────┐  │         │  ┌───────────┐  │
│  │  feature  │  │         │  │  feature  │  │
│  └───────────┘  │         │  └───────────┘  │
└─────────────────┘         └─────────────────┘
```

## Managing Remotes

### List Remotes

```bash
# List remote names
git remote

# List with URLs
git remote -v

# Output:
# origin  git@github.com:user/repo.git (fetch)
# origin  git@github.com:user/repo.git (push)
```

### Add Remote

```bash
# Add remote
git remote add origin git@github.com:user/repo.git

# Add upstream (for forks)
git remote add upstream git@github.com:original/repo.git

# Add multiple remotes
git remote add github git@github.com:user/repo.git
git remote add gitlab git@gitlab.com:user/repo.git
```

### Modify Remote

```bash
# Rename remote
git remote rename origin github

# Change remote URL
git remote set-url origin git@github.com:user/new-repo.git

# Change only push URL
git remote set-url --push origin git@github.com:user/repo.git

# Remove remote
git remote remove origin
git remote rm origin
```

### View Remote Info

```bash
# Show remote details
git remote show origin

# Output includes:
# - Fetch/push URLs
# - HEAD branch
# - Remote branches
# - Tracking status
```

## Fetching

Download objects and refs from remote without merging.

```bash
# Fetch from origin
git fetch origin

# Fetch all remotes
git fetch --all

# Fetch specific branch
git fetch origin main

# Fetch and prune deleted branches
git fetch --prune
git fetch -p

# Fetch tags
git fetch --tags
```

### After Fetching

```bash
# See what was fetched
git log origin/main

# Compare local and remote
git log main..origin/main
git diff main origin/main

# Merge manually
git merge origin/main
```

## Pulling

Fetch and merge in one command.

```bash
# Pull current branch
git pull

# Pull from specific remote/branch
git pull origin main

# Pull with rebase (avoid merge commits)
git pull --rebase
git pull -r

# Pull and prune
git pull --prune

# Pull all branches
git pull --all
```

### Pull Strategies

```bash
# Default: merge
git config --global pull.rebase false

# Rebase
git config --global pull.rebase true

# Fast-forward only
git config --global pull.ff only
```

## Pushing

Upload local commits to remote.

```bash
# Push current branch
git push

# Push to specific remote/branch
git push origin main

# Push and set upstream
git push -u origin feature
git push --set-upstream origin feature

# Push all branches
git push --all

# Push tags
git push --tags
git push origin v1.0.0

# Force push (dangerous!)
git push --force
git push -f

# Force with lease (safer)
git push --force-with-lease

# Delete remote branch
git push origin --delete branch-name
```

### Push Configurations

```bash
# Push only current branch
git config --global push.default current

# Push matching branches
git config --global push.default matching

# Auto setup remote tracking
git config --global push.autoSetupRemote true
```

## Tracking Branches

Remote-tracking branches are references to remote branch states.

```bash
# View tracking info
git branch -vv

# Set upstream branch
git branch -u origin/main
git branch --set-upstream-to=origin/main

# Remove upstream
git branch --unset-upstream

# Create tracking branch
git checkout --track origin/feature
git checkout -b feature origin/feature
```

## Forking Workflow

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│   Original Repo                    Your Fork                 │
│   (upstream)                       (origin)                  │
│                                                              │
│   ┌─────────┐    Fork on GitHub    ┌─────────┐              │
│   │ GitHub  │ ──────────────────►  │ GitHub  │              │
│   │original │                      │ your    │              │
│   └────┬────┘                      └────┬────┘              │
│        │                                │                    │
│        │                                │ clone              │
│        │                                ▼                    │
│        │                          ┌─────────┐               │
│        └── upstream ─────────────►│  Local  │◄── origin     │
│                                   │  Repo   │               │
│                                   └─────────┘               │
└─────────────────────────────────────────────────────────────┘
```

### Fork Workflow Steps

```bash
# 1. Fork on GitHub (via web interface)

# 2. Clone your fork
git clone git@github.com:you/repo.git
cd repo

# 3. Add upstream remote
git remote add upstream git@github.com:original/repo.git

# 4. Verify remotes
git remote -v
# origin    git@github.com:you/repo.git (fetch)
# origin    git@github.com:you/repo.git (push)
# upstream  git@github.com:original/repo.git (fetch)
# upstream  git@github.com:original/repo.git (push)

# 5. Create feature branch
git checkout -b feature

# 6. Make changes and push to fork
git add .
git commit -m "Add feature"
git push -u origin feature

# 7. Create Pull Request on GitHub (via web)

# 8. Keep fork updated
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

## Syncing with Remote

### Update Local from Remote

```bash
# Fetch and merge (current branch)
git pull

# Fetch and rebase
git pull --rebase

# Manual: fetch then merge
git fetch origin
git merge origin/main
```

### Update Remote from Local

```bash
# Push changes
git push origin main

# Push new branch
git push -u origin feature
```

### Sync Fork with Upstream

```bash
# Fetch upstream
git fetch upstream

# Merge upstream into local main
git checkout main
git merge upstream/main

# Push to your fork
git push origin main
```

## Practical Scenarios

### Clone and Push Changes

```bash
# Clone repo
git clone git@github.com:user/repo.git
cd repo

# Make changes
echo "change" >> file.txt
git add file.txt
git commit -m "Update file"

# Push
git push origin main
```

### Work with Multiple Remotes

```bash
# Add multiple remotes
git remote add github git@github.com:user/repo.git
git remote add gitlab git@gitlab.com:user/repo.git

# Push to specific remote
git push github main
git push gitlab main

# Push to all remotes
git remote | xargs -L1 git push --all
```

### Recover from Force Push

```bash
# Someone else force pushed, your push is rejected
git fetch origin
git reset --hard origin/main  # Lose local commits!

# Or rebase your commits on top
git fetch origin
git rebase origin/main
git push --force-with-lease
```

## Quick Reference

| Command | Purpose |
|---------|---------|
| `git remote -v` | List remotes |
| `git remote add name url` | Add remote |
| `git remote remove name` | Remove remote |
| `git fetch` | Download from remote |
| `git pull` | Fetch and merge |
| `git pull --rebase` | Fetch and rebase |
| `git push` | Upload to remote |
| `git push -u origin branch` | Push and track |
| `git push --force-with-lease` | Safe force push |
| `git branch -vv` | Show tracking info |

---

**Previous:** [05-merging-rebasing.md](05-merging-rebasing.md) | **Next:** [07-undoing-changes.md](07-undoing-changes.md)
