# Introduction to Git

## What is Version Control?

Version control is a system that records changes to files over time, allowing you to:
- Track history of changes
- Revert to previous versions
- Collaborate with others
- Work on multiple features simultaneously

## Types of Version Control

### Local Version Control

```
┌─────────────────────────────────────┐
│           Your Computer             │
│  ┌─────────────┐  ┌──────────────┐  │
│  │   Files     │  │  Version DB  │  │
│  │             │  │  v1, v2, v3  │  │
│  └─────────────┘  └──────────────┘  │
└─────────────────────────────────────┘
```
Example: RCS

### Centralized Version Control

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│   ┌────────┐     ┌────────────────┐     ┌────────┐          │
│   │Client A│◄───►│ Central Server │◄───►│Client B│          │
│   └────────┘     │  (Repository)  │     └────────┘          │
│                  └────────────────┘                          │
│                          ▲                                   │
│                          │                                   │
│                     ┌────────┐                               │
│                     │Client C│                               │
│                     └────────┘                               │
└─────────────────────────────────────────────────────────────┘
```
Examples: SVN, CVS, Perforce

### Distributed Version Control

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│   ┌──────────────┐                    ┌──────────────┐      │
│   │   Client A   │◄──────────────────►│   Client B   │      │
│   │ (Full Repo)  │                    │ (Full Repo)  │      │
│   └──────┬───────┘                    └───────┬──────┘      │
│          │                                    │              │
│          │        ┌────────────────┐          │              │
│          └───────►│ Remote Server  │◄─────────┘              │
│                   │  (Full Repo)   │                         │
│                   └───────┬────────┘                         │
│                           │                                  │
│                   ┌───────▼────────┐                         │
│                   │   Client C     │                         │
│                   │  (Full Repo)   │                         │
│                   └────────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```
Examples: Git, Mercurial

## What is Git?

Git is a **distributed version control system** created by Linus Torvalds in 2005 for Linux kernel development.

### Key Characteristics

| Feature | Description |
|---------|-------------|
| Distributed | Every clone is a full repository |
| Fast | Most operations are local |
| Branching | Lightweight, easy branching |
| Integrity | SHA-1 checksums for everything |
| Staging Area | Prepare commits precisely |
| Free & Open Source | GPL v2 license |

## Git vs GitHub

| Git | GitHub |
|-----|--------|
| Version control software | Hosting platform for Git repos |
| Runs locally | Cloud-based service |
| Command-line tool | Web interface + API |
| Created by Linus Torvalds | Created by GitHub Inc (Microsoft) |
| Free, open source | Free + paid plans |

### Other Git Hosting Platforms

- **GitLab** - Self-hostable, CI/CD built-in
- **Bitbucket** - Atlassian, integrates with Jira
- **Azure DevOps** - Microsoft, enterprise features
- **Gitea** - Lightweight, self-hosted
- **Codeberg** - Non-profit, community-driven

## Git Architecture

### Three States

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  Working Directory      Staging Area       Repository        │
│  (Working Tree)         (Index)            (.git)            │
│                                                              │
│  ┌─────────────┐       ┌─────────────┐    ┌─────────────┐   │
│  │             │       │             │    │             │   │
│  │   Files     │──────►│   Staged    │───►│  Committed  │   │
│  │             │  add  │   Changes   │    │   History   │   │
│  │             │       │             │    │             │   │
│  └─────────────┘       └─────────────┘    └─────────────┘   │
│                                                              │
│        ◄────────────── checkout ──────────────────          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### File States

| State | Description |
|-------|-------------|
| Untracked | New file, Git doesn't know about it |
| Tracked | File Git is watching |
| Unmodified | Tracked, no changes since last commit |
| Modified | Changed since last commit |
| Staged | Marked for next commit |

### .git Directory

```
.git/
├── HEAD              # Current branch reference
├── config            # Repository configuration
├── description       # Repository description
├── hooks/            # Client/server-side scripts
├── index             # Staging area
├── objects/          # All content (blobs, trees, commits)
│   ├── pack/         # Packed objects
│   └── info/
├── refs/             # Pointers to commits
│   ├── heads/        # Branch references
│   ├── tags/         # Tag references
│   └── remotes/      # Remote tracking branches
└── logs/             # Reference logs
```

## Git Objects

### Types

| Object | Description |
|--------|-------------|
| Blob | File contents |
| Tree | Directory structure |
| Commit | Snapshot + metadata |
| Tag | Named reference to commit |

### Commit Structure

```
┌─────────────────────────────────────────┐
│              Commit Object               │
├─────────────────────────────────────────┤
│  SHA-1: a1b2c3d4e5f6...                 │
│  Tree: 9f8e7d6c5b4a...                  │
│  Parent: 1a2b3c4d5e6f...                │
│  Author: John <john@example.com>        │
│  Date: Mon Jan 15 10:30:00 2024         │
│  Message: Add new feature               │
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│               Tree Object                │
├─────────────────────────────────────────┤
│  blob a1b2c3... README.md               │
│  blob d4e5f6... index.html              │
│  tree 7g8h9i... src/                    │
└─────────────────────────────────────────┘
```

## Why Git for DevOps?

1. **Infrastructure as Code** - Track infrastructure changes
2. **CI/CD Pipelines** - Trigger builds on commits
3. **GitOps** - Git as single source of truth
4. **Collaboration** - Team workflows with branches/PRs
5. **Audit Trail** - Complete history of changes
6. **Rollback** - Easily revert to working state
7. **Automation** - Hooks and GitHub Actions

## Basic Git Workflow

```
1. Clone/Init repository
         ↓
2. Create branch for feature
         ↓
3. Make changes to files
         ↓
4. Stage changes (git add)
         ↓
5. Commit changes (git commit)
         ↓
6. Push to remote (git push)
         ↓
7. Create Pull Request
         ↓
8. Review and Merge
         ↓
9. Pull latest changes (git pull)
```

## Quick Reference

| Term | Description |
|------|-------------|
| Repository | Project folder tracked by Git |
| Commit | Snapshot of changes |
| Branch | Independent line of development |
| Merge | Combine branches |
| Clone | Copy remote repository |
| Push | Upload commits to remote |
| Pull | Download commits from remote |
| HEAD | Current branch/commit pointer |

---

**Next:** [02-installation-config.md](02-installation-config.md)
