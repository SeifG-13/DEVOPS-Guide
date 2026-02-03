# Git & GitHub Best Practices

## Commit Best Practices

### Write Good Commit Messages

```
<type>(<scope>): <subject>

<body>

<footer>
```

#### Conventional Commits

| Type | Purpose |
|------|---------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation |
| `style` | Formatting (no code change) |
| `refactor` | Code restructuring |
| `test` | Adding tests |
| `chore` | Maintenance tasks |
| `perf` | Performance improvement |
| `ci` | CI/CD changes |

#### Examples

```bash
# Good
feat(auth): add password reset functionality
fix(api): handle null response from server
docs(readme): update installation instructions
refactor(utils): simplify date formatting logic

# Bad
fixed stuff
updates
WIP
asdfgh
```

#### Message Guidelines

```
# Subject line (first line)
- 50 characters or less
- Imperative mood ("Add" not "Added")
- No period at end
- Capitalize first letter

# Body (optional)
- Wrap at 72 characters
- Explain what and why, not how
- Separate from subject with blank line

# Footer (optional)
- Reference issues: Fixes #123
- Breaking changes: BREAKING CHANGE: description
```

### Commit Frequency

| Practice | Description |
|----------|-------------|
| Commit often | Small, logical units |
| Atomic commits | One change per commit |
| Working state | Each commit should work |
| Don't commit | Broken code, debug statements |

### What NOT to Commit

```gitignore
# Never commit
- Secrets (API keys, passwords)
- Environment files (.env)
- Dependencies (node_modules)
- Build outputs (dist, build)
- IDE settings (.idea, .vscode)
- OS files (.DS_Store)
- Large binary files
```

## Branching Best Practices

### Naming Conventions

```bash
# Feature branches
feature/user-authentication
feature/JIRA-123-add-login

# Bug fixes
bugfix/fix-header-crash
bugfix/JIRA-456-null-pointer

# Hotfixes
hotfix/security-patch
hotfix/v1.0.1

# Releases
release/v1.0.0
release/2024-q1

# Personal/experimental
experiment/try-new-api
user/john/testing
```

### Branch Guidelines

| Practice | Description |
|----------|-------------|
| Short-lived | Merge within days, not weeks |
| Up to date | Regularly sync with main |
| One purpose | Single feature/fix per branch |
| Delete after merge | Clean up merged branches |

### Protected Branches

```yaml
# main branch rules
- Require pull request reviews (1-2 reviewers)
- Require status checks to pass
- Require branches to be up to date
- Require signed commits (optional)
- No force pushes
- No deletions
```

## Pull Request Best Practices

### Creating PRs

```markdown
## PR Title
- Clear and concise
- Include ticket number if applicable
- Use conventional commit format

## PR Description
- Explain what changes were made
- Explain why changes were needed
- List any breaking changes
- Include screenshots for UI changes
- Link related issues

## Size
- Keep PRs small (< 400 lines)
- Split large changes into smaller PRs
- One feature/fix per PR
```

### PR Template

```markdown
## Summary
Brief description of changes.

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manual testing done

## Checklist
- [ ] Code follows style guide
- [ ] Self-reviewed code
- [ ] Documentation updated
- [ ] No new warnings

## Related Issues
Fixes #123
```

### Code Review Guidelines

**As Reviewer:**
- Be constructive and respectful
- Focus on code, not person
- Explain why, not just what
- Approve when "good enough"
- Use suggestions feature

**As Author:**
- Respond to all comments
- Don't take feedback personally
- Explain decisions when needed
- Request re-review after changes

## Repository Structure

### Standard Files

```
repository/
├── .github/
│   ├── workflows/           # CI/CD
│   ├── ISSUE_TEMPLATE/      # Issue templates
│   ├── PULL_REQUEST_TEMPLATE.md
│   ├── CODEOWNERS
│   └── dependabot.yml
├── src/                     # Source code
├── tests/                   # Tests
├── docs/                    # Documentation
├── scripts/                 # Utility scripts
├── .gitignore
├── .gitattributes
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── CHANGELOG.md
└── CODE_OF_CONDUCT.md
```

### README Template

```markdown
# Project Name

Brief description.

## Features
- Feature 1
- Feature 2

## Installation

```bash
npm install project-name
```

## Usage

```javascript
import { feature } from 'project-name';
```

## Documentation
Link to full docs.

## Contributing
See CONTRIBUTING.md

## License
MIT License
```

### CODEOWNERS

```
# Default owner
* @team-lead

# By directory
/src/frontend/ @frontend-team
/src/backend/ @backend-team
/infrastructure/ @devops-team

# By file type
*.sql @database-team
*.md @docs-team
```

## Security Best Practices

### Secrets Management

```bash
# Never commit secrets
# Use environment variables
# Use GitHub Secrets for CI/CD

# If you accidentally commit a secret:
1. Rotate the secret immediately
2. Remove from history (if possible)
3. Add to .gitignore
```

### Signed Commits

```bash
# Configure GPG signing
git config --global user.signingkey YOUR_KEY_ID
git config --global commit.gpgsign true

# Sign commits
git commit -S -m "Signed commit"
```

### Branch Protection

```
✓ Require pull request reviews
✓ Dismiss stale reviews
✓ Require review from CODEOWNERS
✓ Require status checks
✓ Require signed commits
✓ Include administrators
```

## Workflow Best Practices

### Daily Workflow

```bash
# Start of day
git checkout main
git pull origin main

# Create feature branch
git checkout -b feature/my-feature

# Work in small commits
git add -p  # Stage selectively
git commit -m "feat: add component"

# Stay updated
git fetch origin
git rebase origin/main

# Push and create PR
git push -u origin feature/my-feature
gh pr create
```

### Before Merging

```bash
# Update branch
git fetch origin
git rebase origin/main

# Run tests
npm test

# Check lint
npm run lint

# Squash if needed
git rebase -i origin/main
```

### After Merging

```bash
# Update local main
git checkout main
git pull origin main

# Delete feature branch
git branch -d feature/my-feature

# Prune remote branches
git fetch --prune
```

## CI/CD Best Practices

### Pipeline Structure

```yaml
jobs:
  lint:        # Fast feedback first
  test:        # Run tests
  build:       # Build artifacts
  deploy-staging:
    needs: [lint, test, build]
  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
```

### Caching

```yaml
- uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
```

### Secrets

```yaml
# Use GitHub Secrets
env:
  API_KEY: ${{ secrets.API_KEY }}

# Never echo secrets
- run: echo $API_KEY  # DON'T DO THIS
```

## Quick Reference

### Do's

- ✅ Write descriptive commit messages
- ✅ Keep commits small and focused
- ✅ Create short-lived branches
- ✅ Keep PRs small (< 400 lines)
- ✅ Write tests for changes
- ✅ Review your own code first
- ✅ Keep main always deployable
- ✅ Use branch protection
- ✅ Sign commits (for sensitive repos)

### Don'ts

- ❌ Commit secrets or credentials
- ❌ Force push to shared branches
- ❌ Commit directly to main
- ❌ Leave branches unmerged for weeks
- ❌ Create massive PRs
- ❌ Merge without reviews
- ❌ Ignore CI failures
- ❌ Commit generated files
- ❌ Use vague commit messages

---

**Previous:** [17-github-features.md](17-github-features.md)

---

## Congratulations!

You've completed the Git & GitHub section of the DevOps roadmap. You now understand:

- Git fundamentals and commands
- Branching and merging strategies
- Remote repositories and workflows
- GitHub features and automation
- Best practices for professional development

**Next Topic:** [04-Shell-Scripts](../04-Shell-Scripts/)
