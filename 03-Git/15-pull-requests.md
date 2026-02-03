# Pull Requests

## What is a Pull Request?

A Pull Request (PR) is a request to merge changes from one branch into another with code review.

```
Feature Branch          Pull Request           Main Branch
┌─────────────┐        ┌──────────────┐       ┌─────────────┐
│   Changes   │──────►│   Review     │──────►│  Merged     │
│   Commits   │       │   Discuss    │       │  Changes    │
│             │       │   Approve    │       │             │
└─────────────┘       └──────────────┘       └─────────────┘
```

## Creating Pull Requests

### Via GitHub Web

1. Push branch to GitHub
2. Go to repository
3. Click "Compare & pull request" or "New pull request"
4. Select base and compare branches
5. Fill in title and description
6. Assign reviewers, labels, projects
7. Create pull request

### Via CLI

```bash
# Create PR with defaults
gh pr create

# Create with options
gh pr create --title "Add feature" --body "Description"

# Create as draft
gh pr create --draft

# Create with reviewers
gh pr create --reviewer user1,user2

# Create with labels
gh pr create --label "enhancement,priority:high"

# Create from fork
gh pr create --repo upstream/repo

# Create and open in browser
gh pr create --web
```

## PR Template

Create `.github/PULL_REQUEST_TEMPLATE.md`:

```markdown
## Description
<!-- Describe your changes -->

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Related Issues
<!-- Link to issues: Fixes #123 -->

## Testing
<!-- Describe how you tested -->

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-reviewed my code
- [ ] Added tests
- [ ] Updated documentation
- [ ] No new warnings
```

## Managing Pull Requests

### List PRs

```bash
# List open PRs
gh pr list

# List all PRs
gh pr list --state all

# List your PRs
gh pr list --author @me

# List PRs to review
gh pr list --search "review-requested:@me"

# List merged PRs
gh pr list --state merged
```

### View PRs

```bash
# View PR details
gh pr view 123

# View in browser
gh pr view 123 --web

# View diff
gh pr diff 123

# View files changed
gh pr view 123 --json files
```

### Checkout PR

```bash
# Checkout PR locally
gh pr checkout 123

# Checkout and create branch
gh pr checkout 123 --branch feature-test
```

### Update PR

```bash
# Push more commits to PR branch
git add .
git commit -m "Address review feedback"
git push

# PR updates automatically
```

## Code Review

### Reviewing via Web

1. Go to PR → Files changed
2. Review code changes
3. Add comments (inline or general)
4. Submit review: Approve, Request changes, or Comment

### Reviewing via CLI

```bash
# Start review
gh pr review 123

# Approve
gh pr review 123 --approve

# Request changes
gh pr review 123 --request-changes --body "Please fix..."

# Comment only
gh pr review 123 --comment --body "Looks good overall"
```

### Review Comments

```markdown
# Suggestion (GitHub auto-suggests)
```suggestion
corrected code here
```

# Question
What's the purpose of this function?

# Nitpick (optional fix)
nit: Consider renaming this variable
```

## Merge Strategies

### Merge Commit

Creates merge commit preserving all history.

```bash
gh pr merge 123 --merge
```

```
Before:                 After:
main:    A─B            main:    A─B─────M
          \                        \   /
feature:   C─D                      C─D
```

### Squash Merge

Combines all commits into one.

```bash
gh pr merge 123 --squash
```

```
Before:                 After:
main:    A─B            main:    A─B─CD
          \
feature:   C─D
```

### Rebase Merge

Applies commits on top of base branch.

```bash
gh pr merge 123 --rebase
```

```
Before:                 After:
main:    A─B            main:    A─B─C'─D'
          \
feature:   C─D
```

### Choosing Strategy

| Strategy | When to Use |
|----------|-------------|
| Merge | Preserve complete history |
| Squash | Clean history, many small commits |
| Rebase | Linear history, clean commits |

## Branch Protection Rules

Settings → Branches → Add rule:

| Rule | Purpose |
|------|---------|
| Require pull request reviews | Force code review |
| Require status checks | Must pass CI |
| Require branches to be up to date | Must be current with base |
| Include administrators | Apply rules to admins |
| Restrict pushes | Limit who can push |
| Require signed commits | GPG verification |

## Draft Pull Requests

For work in progress:

```bash
# Create draft
gh pr create --draft

# Convert to ready
gh pr ready 123
```

## Auto-Merge

Enable auto-merge to merge when requirements are met:

```bash
# Enable auto-merge
gh pr merge 123 --auto --squash

# Disable auto-merge
gh pr merge 123 --disable-auto
```

## CODEOWNERS

Auto-assign reviewers based on file paths.

Create `.github/CODEOWNERS`:

```
# Default owner
* @default-team

# Specific paths
/src/frontend/ @frontend-team
/src/backend/ @backend-team
/docs/ @docs-team
*.js @javascript-experts

# Specific file
package.json @package-maintainer
```

## PR Workflow Best Practices

### Before Creating PR

```bash
# Update branch with main
git fetch origin
git rebase origin/main

# Run tests locally
npm test

# Check lint
npm run lint

# Push
git push --force-with-lease
```

### Good PR Description

```markdown
## Summary
Brief description of changes.

## Changes
- Added user authentication
- Updated API endpoints
- Fixed login bug

## Testing
- [ ] Unit tests pass
- [ ] Manual testing done
- [ ] Tested on staging

## Screenshots
[If applicable]

Fixes #123
```

### During Review

- Respond to all comments
- Push fixes as new commits (easier to re-review)
- Mark conversations as resolved
- Request re-review when ready

## Quick Reference

| Command | Purpose |
|---------|---------|
| `gh pr create` | Create PR |
| `gh pr list` | List PRs |
| `gh pr view 123` | View PR |
| `gh pr checkout 123` | Checkout PR |
| `gh pr diff 123` | View PR diff |
| `gh pr review 123 --approve` | Approve PR |
| `gh pr merge 123` | Merge PR |
| `gh pr merge --squash` | Squash merge |
| `gh pr close 123` | Close PR |
| `gh pr ready 123` | Mark ready |

---

**Previous:** [14-github-basics.md](14-github-basics.md) | **Next:** [16-github-actions.md](16-github-actions.md)
