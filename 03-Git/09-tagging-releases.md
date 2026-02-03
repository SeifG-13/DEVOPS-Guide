# Tagging & Releases

## What are Tags?

Tags are references that point to specific commits, typically used for release versions.

```
Tags mark specific points in history:

    v1.0.0        v1.1.0        v2.0.0
       │             │             │
       ▼             ▼             ▼
───○───○───○───○───○───○───○───○───○───○─── main
       │
    Release
```

## Types of Tags

### Lightweight Tags

Simple pointer to a commit (like a branch that doesn't move).

```bash
# Create lightweight tag
git tag v1.0.0

# Tag specific commit
git tag v1.0.0 abc123
```

### Annotated Tags

Full objects with metadata (author, date, message). Recommended for releases.

```bash
# Create annotated tag
git tag -a v1.0.0 -m "Release version 1.0.0"

# Tag specific commit
git tag -a v1.0.0 -m "Release version 1.0.0" abc123

# Create with editor
git tag -a v1.0.0
```

### Signed Tags

Annotated tags signed with GPG.

```bash
# Create signed tag
git tag -s v1.0.0 -m "Signed release version 1.0.0"

# Verify signed tag
git tag -v v1.0.0
```

## Managing Tags

### Listing Tags

```bash
# List all tags
git tag

# List with pattern
git tag -l "v1.*"
git tag --list "v2.0.*"

# List with annotations
git tag -n
git tag -n1    # Show 1 line of message
git tag -n5    # Show 5 lines

# Sort tags
git tag --sort=-version:refname   # Newest first
git tag --sort=version:refname    # Oldest first
```

### Viewing Tags

```bash
# Show tag data
git show v1.0.0

# See commit tagged
git rev-parse v1.0.0
```

### Deleting Tags

```bash
# Delete local tag
git tag -d v1.0.0

# Delete remote tag
git push origin --delete v1.0.0
git push origin :refs/tags/v1.0.0

# Delete multiple local tags
git tag -d v1.0.0 v1.0.1 v1.0.2
```

### Renaming Tags

```bash
# Create new tag pointing to old
git tag new-tag old-tag

# Delete old tag
git tag -d old-tag

# Update remote
git push origin new-tag :old-tag
```

## Sharing Tags

### Push Tags to Remote

```bash
# Push single tag
git push origin v1.0.0

# Push all tags
git push --tags
git push origin --tags

# Push only annotated tags
git push --follow-tags
```

### Fetch Tags

```bash
# Fetch all tags
git fetch --tags

# Fetch specific tag
git fetch origin tag v1.0.0
```

## Checking Out Tags

```bash
# Checkout tag (detached HEAD)
git checkout v1.0.0

# Create branch from tag
git checkout -b release-1.0 v1.0.0

# See files at tag
git show v1.0.0:path/to/file
```

## Semantic Versioning

Standard version numbering: **MAJOR.MINOR.PATCH**

```
v1.2.3
│ │ │
│ │ └── PATCH: Bug fixes, backwards compatible
│ └──── MINOR: New features, backwards compatible
└────── MAJOR: Breaking changes
```

### Examples

| Version | Change Type |
|---------|-------------|
| 1.0.0 | Initial release |
| 1.0.1 | Bug fix |
| 1.1.0 | New feature |
| 2.0.0 | Breaking change |

### Pre-release Versions

```
1.0.0-alpha
1.0.0-alpha.1
1.0.0-beta
1.0.0-beta.2
1.0.0-rc.1
1.0.0
```

## GitHub Releases

### Creating Release via Web

1. Go to repository → Releases
2. Click "Create a new release"
3. Choose/create tag
4. Add release title and notes
5. Attach binary files (optional)
6. Publish release

### Creating Release via CLI (gh)

```bash
# Create release
gh release create v1.0.0 --title "Release 1.0.0" --notes "Release notes here"

# Create with auto-generated notes
gh release create v1.0.0 --generate-notes

# Create draft release
gh release create v1.0.0 --draft

# Create pre-release
gh release create v1.0.0-beta --prerelease

# Upload assets
gh release create v1.0.0 ./dist/app.zip ./dist/app.tar.gz

# List releases
gh release list

# View release
gh release view v1.0.0

# Download release assets
gh release download v1.0.0

# Delete release
gh release delete v1.0.0
```

### Release Notes

```markdown
## What's New

### Features
- Added user authentication (#123)
- New dashboard layout (#456)

### Bug Fixes
- Fixed login error (#789)
- Resolved memory leak (#012)

### Breaking Changes
- Removed deprecated API endpoints

### Contributors
@user1, @user2, @user3
```

## Automating Releases

### GitHub Actions Release

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: npm run build

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            dist/*.zip
            dist/*.tar.gz
          generate_release_notes: true
```

### Semantic Release

```bash
# Install
npm install -D semantic-release

# Configure in package.json
{
  "release": {
    "branches": ["main"]
  }
}

# Commit message convention triggers versioning
# feat: -> minor version
# fix: -> patch version
# BREAKING CHANGE: -> major version
```

## Practical Workflows

### Release Workflow

```bash
# 1. Ensure main is up to date
git checkout main
git pull origin main

# 2. Create release branch (optional)
git checkout -b release/v1.0.0

# 3. Update version numbers
# Edit package.json, version files, etc.
git commit -am "Bump version to 1.0.0"

# 4. Merge to main
git checkout main
git merge release/v1.0.0

# 5. Create tag
git tag -a v1.0.0 -m "Release version 1.0.0"

# 6. Push
git push origin main
git push origin v1.0.0

# 7. Create GitHub release
gh release create v1.0.0 --generate-notes
```

### Hotfix with Tags

```bash
# 1. Create hotfix from tag
git checkout -b hotfix/v1.0.1 v1.0.0

# 2. Fix issue
git commit -am "Fix critical bug"

# 3. Update version
git commit -am "Bump version to 1.0.1"

# 4. Tag and push
git tag -a v1.0.1 -m "Hotfix release"
git push origin hotfix/v1.0.1 --tags

# 5. Merge back to main
git checkout main
git merge hotfix/v1.0.1
git push origin main
```

## Quick Reference

| Command | Purpose |
|---------|---------|
| `git tag v1.0.0` | Create lightweight tag |
| `git tag -a v1.0.0 -m "msg"` | Create annotated tag |
| `git tag -s v1.0.0 -m "msg"` | Create signed tag |
| `git tag` | List tags |
| `git tag -l "v1.*"` | List matching tags |
| `git show v1.0.0` | Show tag details |
| `git tag -d v1.0.0` | Delete local tag |
| `git push origin v1.0.0` | Push tag to remote |
| `git push --tags` | Push all tags |
| `git push origin --delete v1.0.0` | Delete remote tag |
| `git checkout v1.0.0` | Checkout tag |
| `gh release create v1.0.0` | Create GitHub release |

---

**Previous:** [08-stashing.md](08-stashing.md) | **Next:** [10-git-workflows.md](10-git-workflows.md)
