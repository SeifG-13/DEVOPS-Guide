# Version Control Integration

## Branching Strategies

```
┌─────────────────────────────────────────────────────────────────┐
│                   Branching Strategies                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Strategy          Best For                                    │
│   ─────────────────────────────────────────────────────         │
│   Git Flow          Scheduled releases, versioned software     │
│   GitHub Flow       Continuous deployment, web apps            │
│   Trunk-Based       Mature teams, high deployment frequency    │
│   GitLab Flow       Mix of Git Flow and GitHub Flow            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Git Flow

Traditional branching model with multiple long-lived branches.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Git Flow                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   main (production)                                             │
│   ────●────────────────●────────────────●──────────────►       │
│        \              /                /                        │
│         \   release  /                /                         │
│          \    ●─────●                /                          │
│           \  /       \              /                           │
│   develop  \/         \            /                            │
│   ──●──●──●──●──●──●──●──●──●──●──●──●──●──●──────────►        │
│      \    /     \    /                                          │
│       \  /       \  /                                           │
│   feature/login   feature/api                                   │
│      ●──●──●        ●──●                                        │
│                                                                  │
│   Branches:                                                     │
│   • main      - Production code                                 │
│   • develop   - Integration branch                              │
│   • feature/* - New features                                    │
│   • release/* - Release preparation                             │
│   • hotfix/*  - Production fixes                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Git Flow Commands

```bash
# Initialize Git Flow
git flow init

# Start a feature
git flow feature start login

# Finish a feature
git flow feature finish login

# Start a release
git flow release start 1.0.0

# Finish a release
git flow release finish 1.0.0

# Start a hotfix
git flow hotfix start fix-bug

# Finish a hotfix
git flow hotfix finish fix-bug
```

## GitHub Flow

Simple model for continuous deployment.

```
┌─────────────────────────────────────────────────────────────────┐
│                      GitHub Flow                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   main (always deployable)                                      │
│   ────●────●────●────●────●────●────●────●──────────────►      │
│        \       /      \       /                                 │
│         \     /        \     /                                  │
│   feature ●──●    feature ●──●                                  │
│                                                                  │
│   Workflow:                                                     │
│   1. Create branch from main                                    │
│   2. Add commits                                                │
│   3. Open Pull Request                                          │
│   4. Review and discuss                                         │
│   5. Deploy and test                                            │
│   6. Merge to main                                              │
│                                                                  │
│   Rules:                                                        │
│   • main is always deployable                                   │
│   • Branch off main for any change                              │
│   • Use descriptive branch names                                │
│   • Open PR early for discussion                                │
│   • Merge only after approval                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Trunk-Based Development

All developers commit to a single branch (trunk/main).

```
┌─────────────────────────────────────────────────────────────────┐
│                  Trunk-Based Development                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   main (trunk)                                                  │
│   ────●──●──●──●──●──●──●──●──●──●──●──●──●──●──────────►      │
│        \   /  \   /  \   /                                      │
│         \ /    \ /    \ /                                       │
│          ●      ●      ●  (short-lived feature branches)       │
│                                                                  │
│   Characteristics:                                              │
│   • Very short-lived branches (< 1 day)                        │
│   • Small, frequent commits                                     │
│   • Feature flags for incomplete features                       │
│   • High test automation required                               │
│   • Continuous integration essential                            │
│                                                                  │
│   Variations:                                                   │
│   • Direct commits to trunk                                     │
│   • Short-lived feature branches                                │
│   • Release branches cut from trunk                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Pull Request Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                   Pull Request Workflow                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Developer                    CI/CD System                     │
│       │                             │                           │
│       │  1. Create branch           │                           │
│       │  2. Make changes            │                           │
│       │  3. Push to remote          │                           │
│       │                             │                           │
│       │  4. Open Pull Request       │                           │
│       │─────────────────────────────│                           │
│       │                             │                           │
│       │                    5. Trigger CI Pipeline               │
│       │                       • Build                           │
│       │                       • Test                            │
│       │                       • Lint                            │
│       │                       • Security scan                   │
│       │                             │                           │
│       │◄────────────────────────────│                           │
│       │     6. Report status        │                           │
│       │                             │                           │
│   Reviewer                          │                           │
│       │  7. Code review             │                           │
│       │  8. Approve/Request changes │                           │
│       │                             │                           │
│       │  9. Merge PR                │                           │
│       │─────────────────────────────│                           │
│       │                             │                           │
│       │                   10. Trigger CD Pipeline               │
│       │                       • Deploy                          │
│       │                             │                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Branch Protection Rules

### GitHub Branch Protection

```yaml
# Settings → Branches → Branch protection rules

Required checks:
  - Require pull request reviews
    - Required approving reviews: 2
    - Dismiss stale PR approvals
  - Require status checks
    - build
    - test
    - lint
  - Require branches to be up to date
  - Include administrators
  - Restrict who can push
```

### Protected Branch Settings

| Setting | Purpose |
|---------|---------|
| Require PR | No direct pushes |
| Require reviews | Code review before merge |
| Require status checks | CI must pass |
| Require up-to-date | Branch must be current |
| Include admins | Rules apply to everyone |

## Commit Conventions

### Conventional Commits

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Commit Types

| Type | Description |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation |
| `style` | Formatting |
| `refactor` | Code restructuring |
| `test` | Adding tests |
| `chore` | Maintenance |
| `ci` | CI/CD changes |

### Examples

```bash
feat(auth): add login with Google OAuth

fix(api): handle null response from external service

docs(readme): update installation instructions

ci(jenkins): add parallel test execution

chore(deps): upgrade lodash to 4.17.21
```

## Semantic Versioning

```
MAJOR.MINOR.PATCH

1.4.2
│ │ │
│ │ └── Patch: Bug fixes (backwards compatible)
│ └──── Minor: New features (backwards compatible)
└────── Major: Breaking changes
```

### Version Examples

| Change | Version |
|--------|---------|
| Initial release | 1.0.0 |
| Bug fix | 1.0.1 |
| New feature | 1.1.0 |
| Breaking change | 2.0.0 |
| Pre-release | 2.0.0-beta.1 |

## Tagging Releases

```bash
# Create annotated tag
git tag -a v1.0.0 -m "Release version 1.0.0"

# Push tags
git push origin v1.0.0
git push origin --tags

# List tags
git tag -l "v1.*"

# Delete tag
git tag -d v1.0.0
git push origin --delete v1.0.0
```

## CI/CD Triggers

### Branch-Based Triggers

```yaml
# GitHub Actions
on:
  push:
    branches:
      - main
      - 'release/**'
  pull_request:
    branches:
      - main
```

```groovy
// Jenkinsfile
triggers {
    pollSCM('H/5 * * * *')
}

when {
    branch 'main'
}
```

### Tag-Based Triggers

```yaml
# GitHub Actions
on:
  push:
    tags:
      - 'v*'
```

## Monorepo vs Multi-Repo

```
┌─────────────────────────────────────────────────────────────────┐
│                 Monorepo vs Multi-Repo                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Monorepo                         Multi-Repo                   │
│   ────────────────────────         ────────────────────────     │
│   my-company/                      my-company/frontend          │
│   ├── frontend/                    my-company/backend           │
│   ├── backend/                     my-company/shared            │
│   ├── shared/                                                   │
│   └── infrastructure/              Pros:                        │
│                                    • Clear ownership            │
│   Pros:                            • Independent releases       │
│   • Atomic changes                 • Simpler CI/CD              │
│   • Easier refactoring                                          │
│   • Shared tooling                 Cons:                        │
│                                    • Cross-repo changes hard    │
│   Cons:                            • Dependency management      │
│   • Large repository               • Code duplication           │
│   • Complex CI/CD                                               │
│   • Access control harder                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Quick Reference

### Branching Strategies

| Strategy | Branches | Best For |
|----------|----------|----------|
| Git Flow | main, develop, feature, release, hotfix | Versioned releases |
| GitHub Flow | main, feature | Continuous deployment |
| Trunk-Based | main (trunk) | High-frequency deployment |

### PR Best Practices

| Practice | Description |
|----------|-------------|
| Small PRs | Easier to review |
| Descriptive titles | Clear purpose |
| Link issues | Traceability |
| Include tests | Verify changes |
| Self-review first | Catch obvious issues |

---

**Previous:** [01-introduction.md](01-introduction.md) | **Next:** [03-jenkins-installation.md](03-jenkins-installation.md)
