# Git Workflows

## Overview

A Git workflow is a recipe for how to use Git effectively in a team.

## Git Flow

Traditional workflow with multiple long-running branches.

```
┌─────────────────────────────────────────────────────────────────┐
│                         Git Flow                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  main      ●───────────────●───────────────────────●            │
│            │               ▲                       ▲             │
│            │               │                       │             │
│  release   │         ●─────●                 ●─────●            │
│            │        ╱      │                ╱      │             │
│  develop   ●───●───●───────●───●───●───●───●───────●            │
│             ╲     ╱             ╲ ╱   ╲                          │
│  feature     ●───●               ●     ●───●                    │
│                                                                  │
│  hotfix          ●───●                                          │
│                      └──────────────────────► main               │
└─────────────────────────────────────────────────────────────────┘
```

### Branches

| Branch | Purpose | Base | Merges To |
|--------|---------|------|-----------|
| main | Production releases | - | - |
| develop | Integration branch | main | release, main |
| feature/* | New features | develop | develop |
| release/* | Release preparation | develop | main, develop |
| hotfix/* | Urgent production fixes | main | main, develop |

### Git Flow Commands

```bash
# Initialize Git Flow
git flow init

# Start feature
git flow feature start my-feature
# Creates: feature/my-feature from develop

# Finish feature
git flow feature finish my-feature
# Merges to develop, deletes branch

# Start release
git flow release start 1.0.0
# Creates: release/1.0.0 from develop

# Finish release
git flow release finish 1.0.0
# Merges to main and develop, creates tag

# Start hotfix
git flow hotfix start fix-bug
# Creates: hotfix/fix-bug from main

# Finish hotfix
git flow hotfix finish fix-bug
# Merges to main and develop, creates tag
```

### Manual Git Flow

```bash
# Feature
git checkout develop
git checkout -b feature/my-feature
# ... work ...
git checkout develop
git merge feature/my-feature
git branch -d feature/my-feature

# Release
git checkout develop
git checkout -b release/1.0.0
# ... final fixes ...
git checkout main
git merge release/1.0.0
git tag -a v1.0.0 -m "Release 1.0.0"
git checkout develop
git merge release/1.0.0
git branch -d release/1.0.0

# Hotfix
git checkout main
git checkout -b hotfix/critical-fix
# ... fix ...
git checkout main
git merge hotfix/critical-fix
git tag -a v1.0.1 -m "Hotfix 1.0.1"
git checkout develop
git merge hotfix/critical-fix
git branch -d hotfix/critical-fix
```

### When to Use Git Flow

✅ Good for:
- Scheduled release cycles
- Multiple versions in production
- Dedicated release process
- Large teams

❌ Not ideal for:
- Continuous deployment
- Simple projects
- Small teams
- Rapid iteration

## GitHub Flow

Simplified workflow focused on continuous deployment.

```
┌─────────────────────────────────────────────────────────────────┐
│                       GitHub Flow                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  main      ●───●───●───●───●───●───●───●───●                   │
│             ╲   ╲ ╱   ╲ ╱   ╲ ╱                                  │
│  feature     ●───●     ●     ●───●                              │
│              └─ PR ─┘  └─ PR ─┘                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Workflow Steps

1. **Create branch** from main
2. **Add commits** with changes
3. **Open Pull Request**
4. **Review and discuss**
5. **Deploy and test** (from branch)
6. **Merge** to main

### GitHub Flow Commands

```bash
# 1. Update main
git checkout main
git pull origin main

# 2. Create feature branch
git checkout -b feature/add-login

# 3. Work and commit
git add .
git commit -m "Add login feature"

# 4. Push branch
git push -u origin feature/add-login

# 5. Create PR (via GitHub or CLI)
gh pr create --title "Add login feature" --body "Description"

# 6. After review and merge, cleanup
git checkout main
git pull origin main
git branch -d feature/add-login
```

### When to Use GitHub Flow

✅ Good for:
- Continuous deployment
- Simple projects
- Small to medium teams
- Web applications

❌ Not ideal for:
- Multiple versions in production
- Long release cycles
- Complex release processes

## Trunk-Based Development

Everyone commits to main (trunk) frequently.

```
┌─────────────────────────────────────────────────────────────────┐
│                  Trunk-Based Development                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  main    ●───●───●───●───●───●───●───●───●───●                  │
│           ╲ ╱     ╲ ╱                                            │
│  feature   ●       ●    (short-lived, < 1-2 days)               │
│                                                                  │
│  releases        v1.0   v1.1       (tags or branches)           │
│                   │      │                                       │
└─────────────────────────────────────────────────────────────────┘
```

### Key Practices

| Practice | Description |
|----------|-------------|
| Small commits | Merge to main multiple times per day |
| Feature flags | Hide incomplete features |
| Short branches | Live < 1-2 days |
| No long-lived branches | Avoid integration problems |
| CI/CD | Automated testing and deployment |

### Trunk-Based Commands

```bash
# Start work
git checkout main
git pull origin main
git checkout -b quick-feature

# Work (keep it short!)
git add .
git commit -m "Add quick feature"

# Sync with main often
git fetch origin
git rebase origin/main

# Push and create PR
git push -u origin quick-feature
gh pr create

# Merge same day
# (via GitHub)

# Or direct to main (with CI)
git checkout main
git merge quick-feature
git push origin main
```

### When to Use Trunk-Based

✅ Good for:
- Mature CI/CD pipeline
- Experienced teams
- Rapid iteration
- Microservices

❌ Not ideal for:
- Teams new to Git
- Poor test coverage
- Complex features
- Regulated industries (sometimes)

## Forking Workflow

Contributors work in their own forks.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Forking Workflow                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Upstream (original)     ●───●───●───●───●                      │
│                              ▲       ▲                           │
│                              │ PR    │ PR                        │
│  Your Fork               ●───●───●   │                          │
│                          ╲   ╱       │                           │
│  Local                    ●───●──────●                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Forking Workflow Steps

```bash
# 1. Fork on GitHub

# 2. Clone your fork
git clone git@github.com:you/repo.git
cd repo

# 3. Add upstream
git remote add upstream git@github.com:original/repo.git

# 4. Create branch
git checkout -b feature

# 5. Work
git commit -am "Add feature"

# 6. Push to your fork
git push origin feature

# 7. Create PR to upstream (via GitHub)

# 8. Keep fork updated
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

### When to Use Forking

✅ Good for:
- Open source projects
- Untrusted contributors
- Large public projects

❌ Not ideal for:
- Small private teams
- Trusted contributors only

## Comparison

| Workflow | Complexity | Best For |
|----------|------------|----------|
| Git Flow | High | Versioned releases |
| GitHub Flow | Low | Continuous deployment |
| Trunk-Based | Low | Rapid iteration |
| Forking | Medium | Open source |

## Choosing a Workflow

```
                    Do you have versioned releases?
                              │
                    ┌─────────┴─────────┐
                   Yes                  No
                    │                    │
              Use Git Flow      Do you deploy continuously?
                                        │
                              ┌─────────┴─────────┐
                             Yes                  No
                              │                    │
                    Is your team experienced?   GitHub Flow
                              │
                    ┌─────────┴─────────┐
                   Yes                  No
                    │                    │
              Trunk-Based         GitHub Flow
```

## Quick Reference

| Workflow | Main Branches | Feature Process |
|----------|---------------|-----------------|
| Git Flow | main, develop | Branch from develop |
| GitHub Flow | main | Branch from main, PR |
| Trunk-Based | main | Very short branches |
| Forking | upstream, origin | Fork, branch, PR |

---

**Previous:** [09-tagging-releases.md](09-tagging-releases.md) | **Next:** [11-advanced-commands.md](11-advanced-commands.md)
