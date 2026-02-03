# GitHub Basics

## What is GitHub?

GitHub is a cloud-based hosting service for Git repositories with collaboration features.

## Core Concepts

```
┌─────────────────────────────────────────────────────────────────┐
│                       GitHub Ecosystem                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Repository        Issues            Pull Requests              │
│   ┌─────────┐      ┌─────────┐       ┌─────────┐                │
│   │  Code   │      │  Bugs   │       │  Code   │                │
│   │ History │      │Features │       │ Review  │                │
│   │  Docs   │      │  Tasks  │       │  Merge  │                │
│   └─────────┘      └─────────┘       └─────────┘                │
│                                                                  │
│   Actions          Projects           Discussions                │
│   ┌─────────┐      ┌─────────┐       ┌─────────┐                │
│   │  CI/CD  │      │ Kanban  │       │Community│                │
│   │Automation│      │ Boards │       │   Q&A   │                │
│   └─────────┘      └─────────┘       └─────────┘                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Repositories

### Creating a Repository

**Via Web:**
1. Click "+" → "New repository"
2. Enter name, description
3. Choose public/private
4. Add README, .gitignore, license
5. Create repository

**Via CLI:**
```bash
# Create repo
gh repo create my-repo --public --description "My project"

# Create and clone
gh repo create my-repo --public --clone

# Create from current directory
cd my-project
gh repo create --source=. --public

# Create private repo
gh repo create my-repo --private
```

### Repository Structure

```
my-repo/
├── .github/
│   ├── workflows/       # GitHub Actions
│   ├── ISSUE_TEMPLATE/  # Issue templates
│   ├── PULL_REQUEST_TEMPLATE.md
│   ├── CODEOWNERS
│   └── dependabot.yml
├── src/                 # Source code
├── docs/                # Documentation
├── tests/               # Tests
├── .gitignore
├── README.md
├── LICENSE
└── CONTRIBUTING.md
```

### Repository Settings

- **General:** Name, visibility, features
- **Branches:** Protection rules, default branch
- **Webhooks:** External integrations
- **Secrets:** Encrypted environment variables
- **Actions:** CI/CD permissions

## GitHub CLI (gh)

```bash
# Install
# macOS
brew install gh

# Windows
winget install GitHub.cli

# Authenticate
gh auth login

# Common commands
gh repo list                    # List repos
gh repo view                    # View repo
gh repo clone user/repo         # Clone repo
gh issue list                   # List issues
gh pr list                      # List PRs
gh pr create                    # Create PR
gh pr checkout 123              # Checkout PR
gh workflow run                 # Run workflow
gh api /user                    # API calls
```

## Issues

### Creating Issues

**Via Web:**
1. Go to Issues tab
2. Click "New issue"
3. Fill title and description
4. Add labels, assignees, milestone
5. Submit

**Via CLI:**
```bash
# Create issue
gh issue create --title "Bug: login fails" --body "Description here"

# Create with labels
gh issue create --title "Feature request" --label "enhancement,priority:high"

# Create and assign
gh issue create --title "Bug" --assignee @me
```

### Issue Management

```bash
# List issues
gh issue list
gh issue list --label bug
gh issue list --state closed

# View issue
gh issue view 123

# Close issue
gh issue close 123

# Reopen issue
gh issue reopen 123

# Comment on issue
gh issue comment 123 --body "Working on this"
```

### Issue Templates

Create `.github/ISSUE_TEMPLATE/bug_report.md`:

```markdown
---
name: Bug Report
about: Report a bug
title: '[BUG] '
labels: bug
assignees: ''
---

## Description
A clear description of the bug.

## Steps to Reproduce
1. Go to '...'
2. Click on '...'
3. See error

## Expected Behavior
What you expected to happen.

## Screenshots
If applicable, add screenshots.

## Environment
- OS: [e.g., macOS]
- Browser: [e.g., Chrome]
- Version: [e.g., 1.0.0]
```

## Forks and Stars

### Forking

```bash
# Fork via CLI
gh repo fork user/repo

# Fork and clone
gh repo fork user/repo --clone

# Sync fork with upstream
gh repo sync owner/repo
```

### Stars

```bash
# Star a repo
gh api -X PUT /user/starred/owner/repo

# List starred repos
gh api /user/starred --jq '.[].full_name'
```

## Collaborators

```bash
# Add collaborator
gh api repos/{owner}/{repo}/collaborators/{username} -X PUT

# List collaborators
gh api repos/{owner}/{repo}/collaborators --jq '.[].login'
```

## GitHub Organizations

Organizations are shared accounts for teams.

### Features
- Team management
- Access control
- Billing management
- Audit logs
- SAML SSO (Enterprise)

### Team Permissions

| Permission | Description |
|------------|-------------|
| Read | Clone, pull |
| Triage | Manage issues/PRs |
| Write | Push to non-protected branches |
| Maintain | Manage repo without admin access |
| Admin | Full access |

## Notifications

### Watching Repos

- **Not watching:** Only participating and @mentions
- **Watching:** All activity
- **Ignoring:** Never notified

```bash
# Watch repo
gh api -X PUT /repos/{owner}/{repo}/subscription -f subscribed=true

# Unwatch
gh api -X DELETE /repos/{owner}/{repo}/subscription
```

## GitHub Flavored Markdown

### Features

```markdown
# Heading 1
## Heading 2

**bold** and *italic*

- [x] Task list item (completed)
- [ ] Task list item (incomplete)

| Column 1 | Column 2 |
|----------|----------|
| Value 1  | Value 2  |

```javascript
code block with syntax highlighting
```

@username mention
#123 issue reference
```

### Useful Shortcuts

| Feature | Syntax |
|---------|--------|
| Mention user | `@username` |
| Reference issue | `#123` |
| Reference PR | `#456` |
| Reference commit | `abc1234` |
| Close issue from PR | `Fixes #123` |

## Quick Reference

| Command | Purpose |
|---------|---------|
| `gh repo create` | Create repository |
| `gh repo clone` | Clone repository |
| `gh issue create` | Create issue |
| `gh issue list` | List issues |
| `gh pr create` | Create pull request |
| `gh pr list` | List pull requests |
| `gh auth login` | Authenticate |
| `gh api` | Make API calls |

---

**Previous:** [13-gitignore.md](13-gitignore.md) | **Next:** [15-pull-requests.md](15-pull-requests.md)
