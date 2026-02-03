# Git Hooks

## What are Git Hooks?

Hooks are scripts that run automatically at certain points in Git workflow.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Git Hooks                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  commit-msg    pre-commit    post-commit    pre-push            │
│       │             │              │             │               │
│       │             │              │             │               │
│       ▼             ▼              ▼             ▼               │
│    Validate     Run tests     Notify        Build &             │
│    message      & lint        team          deploy              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Hook Location

```bash
# Hooks are stored in
.git/hooks/

# Default hooks (samples)
ls .git/hooks/
# applypatch-msg.sample
# commit-msg.sample
# post-update.sample
# pre-commit.sample
# pre-push.sample
# pre-rebase.sample
# prepare-commit-msg.sample
# update.sample
```

## Client-Side Hooks

### Pre-Commit

Runs before commit is created. Exit non-zero to abort.

```bash
#!/bin/sh
# .git/hooks/pre-commit

# Run linter
npm run lint
if [ $? -ne 0 ]; then
    echo "Linting failed. Please fix errors."
    exit 1
fi

# Run tests
npm test
if [ $? -ne 0 ]; then
    echo "Tests failed. Commit aborted."
    exit 1
fi

# Check for debug statements
if git diff --cached | grep -E "console\.log|debugger" > /dev/null; then
    echo "Error: Commit contains debug statements"
    exit 1
fi

exit 0
```

### Prepare-Commit-Msg

Modify commit message before editor opens.

```bash
#!/bin/sh
# .git/hooks/prepare-commit-msg

COMMIT_MSG_FILE=$1
COMMIT_SOURCE=$2

# Add branch name to commit message
BRANCH_NAME=$(git symbolic-ref --short HEAD)
if [ -n "$BRANCH_NAME" ]; then
    echo "[$BRANCH_NAME] $(cat $COMMIT_MSG_FILE)" > $COMMIT_MSG_FILE
fi
```

### Commit-Msg

Validate commit message format.

```bash
#!/bin/sh
# .git/hooks/commit-msg

COMMIT_MSG_FILE=$1
COMMIT_MSG=$(cat $COMMIT_MSG_FILE)

# Check message length
if [ ${#COMMIT_MSG} -lt 10 ]; then
    echo "Error: Commit message too short (min 10 characters)"
    exit 1
fi

# Check conventional commits format
if ! echo "$COMMIT_MSG" | grep -qE "^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .+"; then
    echo "Error: Commit message must follow Conventional Commits format"
    echo "Example: feat(auth): add login feature"
    exit 1
fi

exit 0
```

### Post-Commit

Runs after commit is created. Cannot abort.

```bash
#!/bin/sh
# .git/hooks/post-commit

# Show commit summary
git log -1 --stat

# Send notification
echo "Commit created: $(git log -1 --pretty=format:'%h - %s')"
```

### Pre-Push

Runs before push. Exit non-zero to abort.

```bash
#!/bin/sh
# .git/hooks/pre-push

# Run full test suite
npm run test:all
if [ $? -ne 0 ]; then
    echo "Tests failed. Push aborted."
    exit 1
fi

# Prevent push to main directly
current_branch=$(git symbolic-ref HEAD | sed -e 's,.*/\(.*\),\1,')
if [ "$current_branch" = "main" ]; then
    echo "Direct push to main is not allowed. Use a PR."
    exit 1
fi

exit 0
```

### Pre-Rebase

```bash
#!/bin/sh
# .git/hooks/pre-rebase

# Prevent rebase on main
if [ "$(git symbolic-ref --short HEAD)" = "main" ]; then
    echo "Cannot rebase main branch"
    exit 1
fi
```

## Server-Side Hooks

### Pre-Receive

Runs when receiving push. Can reject entire push.

```bash
#!/bin/sh
# hooks/pre-receive (on server)

while read oldrev newrev refname; do
    # Reject pushes to main that aren't fast-forward
    if [ "$refname" = "refs/heads/main" ]; then
        if ! git merge-base --is-ancestor $oldrev $newrev; then
            echo "Non-fast-forward push to main rejected"
            exit 1
        fi
    fi
done
```

### Update

Runs for each ref being updated.

### Post-Receive

Runs after push is complete. Good for deployment.

```bash
#!/bin/sh
# hooks/post-receive (on server)

while read oldrev newrev refname; do
    if [ "$refname" = "refs/heads/main" ]; then
        echo "Deploying to production..."
        GIT_WORK_TREE=/var/www/app git checkout -f
        # Restart services
        systemctl restart app
    fi
done
```

## Installing Hooks

### Manual Installation

```bash
# Create hook file
vim .git/hooks/pre-commit

# Make executable
chmod +x .git/hooks/pre-commit
```

### Sharing Hooks

Hooks in `.git/hooks/` aren't tracked. Solutions:

#### Option 1: Custom hooks directory

```bash
# Create hooks in repo
mkdir .githooks

# Configure git to use it
git config core.hooksPath .githooks

# Team members run same command
```

#### Option 2: Symbolic links

```bash
# Create hooks in repo
mkdir scripts/hooks

# Link to .git/hooks
ln -sf ../../scripts/hooks/pre-commit .git/hooks/pre-commit
```

## Using Husky (Node.js)

```bash
# Install Husky
npm install husky --save-dev

# Initialize
npx husky init

# Add hooks
echo "npm test" > .husky/pre-commit
echo "npx commitlint --edit \$1" > .husky/commit-msg
```

### package.json Setup

```json
{
  "scripts": {
    "prepare": "husky"
  }
}
```

### .husky/pre-commit

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npm run lint
npm test
```

## Using pre-commit (Python)

```bash
# Install
pip install pre-commit

# Create config
cat > .pre-commit-config.yaml << EOF
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json

  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black

  - repo: local
    hooks:
      - id: pytest
        name: pytest
        entry: pytest
        language: system
        types: [python]
        pass_filenames: false
EOF

# Install hooks
pre-commit install

# Run manually
pre-commit run --all-files
```

## Common Hook Scripts

### Prevent Large Files

```bash
#!/bin/sh
# pre-commit

MAX_FILE_SIZE=5242880  # 5MB

for file in $(git diff --cached --name-only); do
    size=$(wc -c < "$file")
    if [ $size -gt $MAX_FILE_SIZE ]; then
        echo "Error: $file exceeds 5MB limit"
        exit 1
    fi
done
```

### Check for Secrets

```bash
#!/bin/sh
# pre-commit

if git diff --cached | grep -iE "(password|secret|api_key|token)\s*=" > /dev/null; then
    echo "Warning: Possible secret in commit"
    exit 1
fi
```

### Auto-Format Code

```bash
#!/bin/sh
# pre-commit

# Format staged files
FILES=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(js|ts)$')
if [ -n "$FILES" ]; then
    npx prettier --write $FILES
    git add $FILES
fi
```

## Quick Reference

| Hook | Trigger | Can Abort |
|------|---------|-----------|
| pre-commit | Before commit | Yes |
| prepare-commit-msg | Before msg editor | Yes |
| commit-msg | After msg entered | Yes |
| post-commit | After commit | No |
| pre-push | Before push | Yes |
| pre-rebase | Before rebase | Yes |
| post-checkout | After checkout | No |
| post-merge | After merge | No |

---

**Previous:** [11-advanced-commands.md](11-advanced-commands.md) | **Next:** [13-gitignore.md](13-gitignore.md)
