# .gitignore

## What is .gitignore?

A `.gitignore` file specifies files and directories that Git should ignore.

## Basic Syntax

```gitignore
# Comment

# Ignore specific file
secret.txt

# Ignore by extension
*.log
*.tmp

# Ignore directory
node_modules/
build/

# Negate (don't ignore)
!important.log

# Wildcard patterns
*.py[cod]     # .pyc, .pyo, .pyd
doc/**/*.pdf  # All PDFs in doc subdirs
```

## Pattern Matching

| Pattern | Description | Example |
|---------|-------------|---------|
| `file.txt` | Specific file | `file.txt` |
| `*.log` | Any .log file | `error.log` |
| `dir/` | Directory | `node_modules/` |
| `/file.txt` | Root only | Only root `file.txt` |
| `**/logs` | Any path | `a/b/logs` |
| `logs/**` | Everything inside | `logs/a/b.txt` |
| `a/**/b` | Zero or more dirs | `a/x/y/b` |
| `*.py[cod]` | Character class | `.pyc, .pyo, .pyd` |
| `!pattern` | Negate | Don't ignore |

## Common .gitignore Patterns

### Node.js

```gitignore
# Dependencies
node_modules/
package-lock.json
yarn.lock

# Build output
dist/
build/

# Environment
.env
.env.local
.env.*.local

# Logs
logs/
*.log
npm-debug.log*

# IDE
.idea/
.vscode/
*.sublime-*

# OS
.DS_Store
Thumbs.db

# Cache
.cache/
.npm/
```

### Python

```gitignore
# Byte-compiled
__pycache__/
*.py[cod]
*$py.class

# Virtual environments
venv/
.venv/
env/
.env/

# Distribution
dist/
build/
*.egg-info/
*.egg

# IDE
.idea/
.vscode/
*.swp

# Testing
.pytest_cache/
.coverage
htmlcov/

# Environment
.env
*.env
```

### Java

```gitignore
# Compiled
*.class
*.jar
*.war
*.ear

# Build directories
target/
build/
out/

# IDE
.idea/
*.iml
.eclipse/
.classpath
.project
.settings/

# Logs
*.log

# Package files
*.jar
```

### Go

```gitignore
# Binaries
*.exe
*.exe~
*.dll
*.so
*.dylib

# Test binary
*.test

# Output
/bin/
/pkg/

# Vendor
vendor/

# IDE
.idea/
.vscode/
```

### General Project

```gitignore
# OS generated
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# IDEs
.idea/
.vscode/
*.sublime-project
*.sublime-workspace
*.swp
*.swo
*~

# Dependencies
node_modules/
vendor/
bower_components/

# Build outputs
dist/
build/
out/
*.o
*.a
*.so

# Logs
logs/
*.log

# Environment and secrets
.env
.env.local
.env.*.local
*.pem
*.key

# Coverage
coverage/
.nyc_output/

# Cache
.cache/
*.cache

# Temporary
tmp/
temp/
*.tmp
*.bak
```

## Global .gitignore

For personal files ignored across all repos.

```bash
# Create global gitignore
touch ~/.gitignore_global

# Configure Git to use it
git config --global core.excludesfile ~/.gitignore_global
```

### ~/.gitignore_global

```gitignore
# OS
.DS_Store
Thumbs.db

# Editors
*.swp
*.swo
*~
.idea/
.vscode/
*.sublime-*

# Personal notes
notes.txt
todo.txt
```

## .gitignore Best Practices

### 1. Ignore Early

Add `.gitignore` at project start, before first commit.

### 2. Don't Ignore Important Files

```gitignore
# Don't add these to .gitignore:
# - Source code
# - Configuration templates
# - Documentation
# - Tests
```

### 3. Use Templates

```bash
# GitHub's collection
# https://github.com/github/gitignore

# Create from template
curl https://raw.githubusercontent.com/github/gitignore/main/Node.gitignore > .gitignore
```

### 4. Check What's Ignored

```bash
# Check if file is ignored
git check-ignore -v filename

# List all ignored files
git status --ignored

# List ignored files in directory
git ls-files --ignored --exclude-standard
```

## Ignoring Already Tracked Files

```bash
# Remove from tracking (keep file)
git rm --cached filename

# Remove directory from tracking
git rm -r --cached directory/

# Then add to .gitignore
echo "filename" >> .gitignore
git add .gitignore
git commit -m "Stop tracking filename"
```

## Force Add Ignored File

```bash
# Add despite .gitignore
git add -f ignored-file.txt
```

## Local Ignore (Not Committed)

For personal ignores that shouldn't be in `.gitignore`:

```bash
# Edit local exclude file
vim .git/info/exclude

# Same syntax as .gitignore
my-local-notes.txt
.my-config
```

## Multiple .gitignore Files

You can have `.gitignore` in subdirectories:

```
project/
├── .gitignore         # Project root patterns
├── src/
│   └── .gitignore     # Source-specific patterns
└── docs/
    └── .gitignore     # Docs-specific patterns
```

## Debug .gitignore

```bash
# Why is this file ignored?
git check-ignore -v path/to/file

# Output:
# .gitignore:5:*.log    logs/error.log

# Check all ignored files
git status --ignored --short
```

## Quick Reference

| Pattern | Matches |
|---------|---------|
| `*.log` | All .log files |
| `logs/` | logs directory |
| `/logs/` | logs in root only |
| `**/logs` | logs anywhere |
| `!keep.log` | Don't ignore keep.log |
| `*.py[cod]` | .pyc, .pyo, .pyd |
| `doc/**/*.pdf` | PDFs in doc subdirs |

| Command | Purpose |
|---------|---------|
| `git check-ignore -v file` | Check why ignored |
| `git status --ignored` | List ignored files |
| `git rm --cached file` | Stop tracking file |
| `git add -f file` | Force add ignored |

---

**Previous:** [12-hooks.md](12-hooks.md) | **Next:** [14-github-basics.md](14-github-basics.md)
