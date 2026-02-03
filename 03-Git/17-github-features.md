# GitHub Features

## GitHub Pages

Free static site hosting from your repository.

### Enable Pages

1. Settings → Pages
2. Select source branch
3. Choose folder (root or /docs)
4. Save

### Configuration

```yaml
# _config.yml (Jekyll)
title: My Project
description: Project documentation
theme: jekyll-theme-cayman
```

### Custom Domain

1. Add CNAME file with domain name
2. Configure DNS (A/CNAME records)
3. Enable HTTPS in settings

### Deploy with Actions

```yaml
# .github/workflows/pages.yml
name: Deploy to Pages

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: npm run build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: dist/
      - uses: actions/deploy-pages@v4
```

## GitHub Packages

Package registry integrated with GitHub.

### Package Types

| Registry | Package Type |
|----------|--------------|
| npm | JavaScript |
| Maven | Java |
| NuGet | .NET |
| RubyGems | Ruby |
| Docker | Containers |

### Publish npm Package

```yaml
# .github/workflows/publish.yml
name: Publish Package

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://npm.pkg.github.com'
      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Publish Docker Image

```yaml
name: Docker

on:
  push:
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.ref_name }}
```

## GitHub Codespaces

Cloud development environments.

### Configuration

```json
// .devcontainer/devcontainer.json
{
  "name": "My Project",
  "image": "mcr.microsoft.com/devcontainers/javascript-node:20",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {}
  },
  "postCreateCommand": "npm install",
  "customizations": {
    "vscode": {
      "extensions": ["dbaeumer.vscode-eslint"]
    }
  },
  "forwardPorts": [3000]
}
```

### Usage

```bash
# Create codespace
gh codespace create

# List codespaces
gh codespace list

# Open in VS Code
gh codespace code

# SSH to codespace
gh codespace ssh
```

## Dependabot

Automated dependency updates and security alerts.

### Configuration

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    labels:
      - "dependencies"
    reviewers:
      - "team-lead"

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "daily"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### Security Updates

Enable in Settings → Security → Dependabot security updates

## GitHub Security

### Secret Scanning

Automatically detects exposed secrets.

Enable in Settings → Security → Secret scanning

### Code Scanning

Static analysis for vulnerabilities.

```yaml
# .github/workflows/codeql.yml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 1'

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: javascript
      - uses: github/codeql-action/analyze@v3
```

### Branch Protection

Settings → Branches → Add rule:

```
Pattern: main
✓ Require pull request reviews
✓ Require status checks to pass
✓ Require branches to be up to date
✓ Require signed commits
✓ Do not allow bypassing
```

## GitHub Projects

Project management with Kanban boards.

### Board Views

| View | Purpose |
|------|---------|
| Table | Spreadsheet-like |
| Board | Kanban columns |
| Roadmap | Timeline view |

### Custom Fields

- Status
- Priority
- Sprint
- Estimate
- Due date

### Automation

```yaml
# Auto-add issues to project
name: Add to Project

on:
  issues:
    types: [opened]

jobs:
  add:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/add-to-project@v0.5.0
        with:
          project-url: https://github.com/orgs/org/projects/1
          github-token: ${{ secrets.PROJECT_TOKEN }}
```

## GitHub Discussions

Community forum for your project.

### Categories

| Category | Use For |
|----------|---------|
| Announcements | Updates, news |
| General | Conversations |
| Ideas | Feature requests |
| Q&A | Questions |
| Show and Tell | Showcases |

## Wiki

Built-in documentation.

### Enable

Settings → General → Features → Wikis

### Clone Wiki

```bash
git clone https://github.com/user/repo.wiki.git
```

## GitHub API

```bash
# REST API via gh
gh api /repos/{owner}/{repo}
gh api /repos/{owner}/{repo}/issues --jq '.[].title'

# Create issue
gh api /repos/{owner}/{repo}/issues -f title="Bug" -f body="Description"

# GraphQL
gh api graphql -f query='
  query {
    repository(owner: "owner", name: "repo") {
      issues(first: 10) {
        nodes { title }
      }
    }
  }
'
```

## Quick Reference

| Feature | Purpose |
|---------|---------|
| Pages | Static site hosting |
| Packages | Package registry |
| Codespaces | Cloud dev environments |
| Dependabot | Dependency updates |
| Actions | CI/CD |
| Projects | Project management |
| Discussions | Community forum |
| Wiki | Documentation |
| Security | Vulnerability detection |

---

**Previous:** [16-github-actions.md](16-github-actions.md) | **Next:** [18-best-practices.md](18-best-practices.md)
