# Installation & Configuration

## Installing Git

### Linux

```bash
# Debian/Ubuntu
sudo apt update
sudo apt install git

# RHEL/CentOS/Fedora
sudo yum install git
sudo dnf install git

# Arch Linux
sudo pacman -S git

# Verify installation
git --version
```

### macOS

```bash
# Using Homebrew
brew install git

# Or install Xcode Command Line Tools
xcode-select --install

# Verify
git --version
```

### Windows

1. Download from [git-scm.com](https://git-scm.com/download/win)
2. Run installer
3. Choose options (recommend defaults)
4. Use Git Bash or integrate with terminal

```powershell
# Using Chocolatey
choco install git

# Using winget
winget install Git.Git

# Verify
git --version
```

## Git Configuration

Git has three configuration levels:

| Level | Flag | Location | Scope |
|-------|------|----------|-------|
| System | `--system` | `/etc/gitconfig` | All users |
| Global | `--global` | `~/.gitconfig` | Current user |
| Local | `--local` | `.git/config` | Current repo |

Priority: Local > Global > System

### Essential Configuration

```bash
# Set your identity (required for commits)
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Verify settings
git config --list
git config user.name
git config user.email
```

### Recommended Settings

```bash
# Default branch name
git config --global init.defaultBranch main

# Default editor
git config --global core.editor "vim"
git config --global core.editor "code --wait"  # VS Code
git config --global core.editor "nano"

# Line endings
# Windows
git config --global core.autocrlf true
# Mac/Linux
git config --global core.autocrlf input

# Color output
git config --global color.ui auto

# Push behavior
git config --global push.default current

# Pull behavior (avoid merge commits)
git config --global pull.rebase true

# Default merge tool
git config --global merge.tool vimdiff

# Credential caching
git config --global credential.helper cache
git config --global credential.helper 'cache --timeout=3600'

# Store credentials permanently (less secure)
git config --global credential.helper store
```

### View Configuration

```bash
# List all settings
git config --list

# List with origin (where it's set)
git config --list --show-origin

# Show specific setting
git config user.name

# Edit config file directly
git config --global --edit
```

## SSH Key Setup

SSH keys allow secure, passwordless authentication with GitHub/GitLab.

### Generate SSH Key

```bash
# Generate Ed25519 key (recommended)
ssh-keygen -t ed25519 -C "your.email@example.com"

# Or RSA (more compatible)
ssh-keygen -t rsa -b 4096 -C "your.email@example.com"

# Follow prompts:
# - Save location (default: ~/.ssh/id_ed25519)
# - Passphrase (recommended for security)
```

### Start SSH Agent

```bash
# Start ssh-agent
eval "$(ssh-agent -s)"

# Add key to agent
ssh-add ~/.ssh/id_ed25519
```

### Add Key to GitHub

```bash
# Copy public key
cat ~/.ssh/id_ed25519.pub
# Or
pbcopy < ~/.ssh/id_ed25519.pub      # macOS
clip < ~/.ssh/id_ed25519.pub        # Windows
xclip -sel clip < ~/.ssh/id_ed25519.pub  # Linux
```

Then:
1. Go to GitHub → Settings → SSH and GPG keys
2. Click "New SSH key"
3. Paste public key
4. Save

### Test SSH Connection

```bash
ssh -T git@github.com
# Hi username! You've successfully authenticated...
```

### SSH Config File

```bash
# ~/.ssh/config

# GitHub
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519

# GitLab
Host gitlab.com
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/id_ed25519_gitlab

# Custom alias
Host myserver
    HostName github.com
    User git
    IdentityFile ~/.ssh/work_key
```

## GPG Signing

Sign commits to verify author identity.

### Generate GPG Key

```bash
# Generate key
gpg --full-generate-key

# Choose:
# - RSA and RSA
# - 4096 bits
# - Expiration (e.g., 1y)
# - Name and email (must match Git config)

# List keys
gpg --list-secret-keys --keyid-format=long
# sec   rsa4096/3AA5C34371567BD2 2024-01-15
#       Key fingerprint = ...
# uid   Your Name <your.email@example.com>

# Export public key (for GitHub)
gpg --armor --export 3AA5C34371567BD2
```

### Configure Git for Signing

```bash
# Set signing key
git config --global user.signingkey 3AA5C34371567BD2

# Sign all commits by default
git config --global commit.gpgsign true

# Sign all tags by default
git config --global tag.gpgsign true
```

### Add GPG Key to GitHub

1. Go to GitHub → Settings → SSH and GPG keys
2. Click "New GPG key"
3. Paste exported public key
4. Save

### Verify Signed Commits

```bash
# Sign a commit
git commit -S -m "Signed commit"

# Verify signature
git log --show-signature

# Verify specific commit
git verify-commit <commit-hash>
```

## .gitconfig File

Example `~/.gitconfig`:

```ini
[user]
    name = Your Name
    email = your.email@example.com
    signingkey = 3AA5C34371567BD2

[init]
    defaultBranch = main

[core]
    editor = vim
    autocrlf = input
    whitespace = fix
    pager = less -R

[color]
    ui = auto

[push]
    default = current
    autoSetupRemote = true

[pull]
    rebase = true

[fetch]
    prune = true

[merge]
    tool = vimdiff
    conflictstyle = diff3

[diff]
    tool = vimdiff
    colorMoved = zebra

[commit]
    gpgsign = true

[alias]
    st = status
    co = checkout
    br = branch
    ci = commit
    lg = log --oneline --graph --decorate
    last = log -1 HEAD
    unstage = reset HEAD --
    undo = reset --soft HEAD~1
    amend = commit --amend --no-edit

[credential]
    helper = cache --timeout=3600

[filter "lfs"]
    clean = git-lfs clean -- %f
    smudge = git-lfs smudge -- %f
    process = git-lfs filter-process
    required = true
```

## Git Aliases

### Create Aliases

```bash
# Short commands
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit

# Useful aliases
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.last "log -1 HEAD"
git config --global alias.unstage "reset HEAD --"
git config --global alias.undo "reset --soft HEAD~1"
git config --global alias.amend "commit --amend --no-edit"
git config --global alias.branches "branch -a"
git config --global alias.remotes "remote -v"
git config --global alias.stashes "stash list"
```

### Using Aliases

```bash
git st          # git status
git co main     # git checkout main
git lg          # Pretty log
git amend       # Amend without editing message
```

## Quick Reference

| Command | Purpose |
|---------|---------|
| `git config --list` | Show all settings |
| `git config --global user.name "Name"` | Set username |
| `git config --global user.email "email"` | Set email |
| `git config --global --edit` | Edit config file |
| `ssh-keygen -t ed25519` | Generate SSH key |
| `ssh -T git@github.com` | Test GitHub connection |

---

**Previous:** [01-introduction.md](01-introduction.md) | **Next:** [03-basic-commands.md](03-basic-commands.md)
