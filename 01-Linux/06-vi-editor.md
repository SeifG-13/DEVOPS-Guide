# Vi/Vim Editor

## Introduction

Vi (Visual Editor) is a text editor available on virtually every Linux system. Vim (Vi IMproved) is an enhanced version with more features.

```bash
# Open file
vi filename.txt
vim filename.txt

# Open file at specific line
vim +10 filename.txt

# Open file and go to pattern
vim +/pattern filename.txt

# Open in read-only mode
vim -R filename.txt
view filename.txt
```

## Vi Modes

```
┌─────────────────────────────────────────────────────────┐
│                     NORMAL MODE                          │
│              (Navigation & Commands)                     │
│                                                          │
│         i, a, o, O          Esc                         │
│              │                │                          │
│              ▼                │                          │
│       ┌─────────────┐         │                          │
│       │ INSERT MODE │─────────┘                          │
│       │  (Typing)   │                                    │
│       └─────────────┘                                    │
│                                                          │
│              :                                           │
│              │                                           │
│              ▼                                           │
│       ┌─────────────┐                                    │
│       │COMMAND MODE │  (Enter/Esc to exit)               │
│       │ (Ex commands)│                                   │
│       └─────────────┘                                    │
│                                                          │
│              v, V, Ctrl+v                                │
│              │                                           │
│              ▼                                           │
│       ┌─────────────┐                                    │
│       │ VISUAL MODE │  (Esc to exit)                     │
│       │ (Selection) │                                    │
│       └─────────────┘                                    │
└─────────────────────────────────────────────────────────┘
```

| Mode | Purpose | Enter | Exit |
|------|---------|-------|------|
| Normal | Navigation, commands | Default / `Esc` | - |
| Insert | Text input | `i`, `a`, `o`, `O` | `Esc` |
| Visual | Text selection | `v`, `V`, `Ctrl+v` | `Esc` |
| Command | Ex commands | `:` | `Enter` or `Esc` |

## Entering Insert Mode

| Key | Action |
|-----|--------|
| `i` | Insert before cursor |
| `I` | Insert at beginning of line |
| `a` | Append after cursor |
| `A` | Append at end of line |
| `o` | Open new line below |
| `O` | Open new line above |
| `s` | Substitute character |
| `S` | Substitute entire line |

## Navigation (Normal Mode)

### Basic Movement

| Key | Movement |
|-----|----------|
| `h` | Left |
| `j` | Down |
| `k` | Up |
| `l` | Right |
| `0` | Beginning of line |
| `^` | First non-blank character |
| `$` | End of line |

### Word Movement

| Key | Movement |
|-----|----------|
| `w` | Next word (start) |
| `W` | Next WORD (space-separated) |
| `e` | Next word (end) |
| `E` | Next WORD (end) |
| `b` | Previous word |
| `B` | Previous WORD |

### Line Movement

| Key | Movement |
|-----|----------|
| `gg` | First line of file |
| `G` | Last line of file |
| `10G` or `:10` | Go to line 10 |
| `Ctrl+g` | Show current line info |

### Screen Movement

| Key | Movement |
|-----|----------|
| `Ctrl+f` | Page forward (down) |
| `Ctrl+b` | Page backward (up) |
| `Ctrl+d` | Half page down |
| `Ctrl+u` | Half page up |
| `H` | Top of screen |
| `M` | Middle of screen |
| `L` | Bottom of screen |

## Editing (Normal Mode)

### Delete

| Key | Action |
|-----|--------|
| `x` | Delete character under cursor |
| `X` | Delete character before cursor |
| `dd` | Delete entire line |
| `D` | Delete from cursor to end of line |
| `dw` | Delete word |
| `d$` | Delete to end of line |
| `d0` | Delete to beginning of line |
| `dG` | Delete to end of file |
| `dgg` | Delete to beginning of file |
| `5dd` | Delete 5 lines |

### Copy (Yank)

| Key | Action |
|-----|--------|
| `yy` | Yank (copy) line |
| `Y` | Yank line |
| `yw` | Yank word |
| `y$` | Yank to end of line |
| `5yy` | Yank 5 lines |

### Paste

| Key | Action |
|-----|--------|
| `p` | Paste after cursor |
| `P` | Paste before cursor |

### Undo / Redo

| Key | Action |
|-----|--------|
| `u` | Undo last change |
| `U` | Undo all changes on line |
| `Ctrl+r` | Redo |
| `.` | Repeat last command |

### Change

| Key | Action |
|-----|--------|
| `r` | Replace single character |
| `R` | Replace mode (overwrite) |
| `cw` | Change word |
| `cc` | Change entire line |
| `C` | Change to end of line |
| `~` | Toggle case |

## Search and Replace

### Search

| Key | Action |
|-----|--------|
| `/pattern` | Search forward |
| `?pattern` | Search backward |
| `n` | Next match |
| `N` | Previous match |
| `*` | Search word under cursor (forward) |
| `#` | Search word under cursor (backward) |

### Search and Replace

```bash
# Replace first occurrence on current line
:s/old/new/

# Replace all occurrences on current line
:s/old/new/g

# Replace all occurrences in entire file
:%s/old/new/g

# Replace with confirmation
:%s/old/new/gc

# Replace in line range (lines 5-10)
:5,10s/old/new/g

# Case insensitive search
:%s/old/new/gi
```

## Visual Mode

| Key | Action |
|-----|--------|
| `v` | Character-wise selection |
| `V` | Line-wise selection |
| `Ctrl+v` | Block (column) selection |

### In Visual Mode

| Key | Action |
|-----|--------|
| `d` | Delete selected |
| `y` | Yank selected |
| `>` | Indent right |
| `<` | Indent left |
| `~` | Toggle case |
| `:` | Command on selection |

## Save and Quit (Command Mode)

| Command | Action |
|---------|--------|
| `:w` | Save (write) |
| `:w filename` | Save as filename |
| `:q` | Quit (fails if unsaved changes) |
| `:q!` | Quit without saving |
| `:wq` | Save and quit |
| `:x` | Save and quit (same as :wq) |
| `ZZ` | Save and quit (Normal mode) |
| `ZQ` | Quit without saving (Normal mode) |
| `:wq!` | Force save and quit |

## Useful Commands

```bash
# Show line numbers
:set number
:set nu

# Hide line numbers
:set nonumber
:set nonu

# Syntax highlighting
:syntax on
:syntax off

# Search highlighting
:set hlsearch
:set nohlsearch
:noh              # Clear current highlighting

# Case insensitive search
:set ignorecase
:set ic

# Show matching brackets
:set showmatch

# Auto indent
:set autoindent
:set ai

# Tab settings
:set tabstop=4
:set shiftwidth=4
:set expandtab    # Use spaces instead of tabs
```

## Multiple Files

```bash
# Open multiple files
vim file1.txt file2.txt

# Navigate between files
:n              # Next file
:N              # Previous file
:first          # First file
:last           # Last file

# List open files
:ls
:buffers

# Switch to buffer
:b2             # Switch to buffer 2
:b filename     # Switch to buffer by name

# Split windows
:split filename     # Horizontal split
:vsplit filename    # Vertical split
Ctrl+w w           # Switch between windows
Ctrl+w h/j/k/l     # Navigate windows
:close             # Close current window
```

## Configuration File

Create/edit `~/.vimrc` for persistent settings:

```bash
" Example .vimrc
set number              " Show line numbers
set relativenumber      " Relative line numbers
syntax on               " Syntax highlighting
set tabstop=4           " Tab width
set shiftwidth=4        " Indent width
set expandtab           " Spaces instead of tabs
set autoindent          " Auto indent
set hlsearch            " Highlight search
set incsearch           " Incremental search
set ignorecase          " Case insensitive search
set smartcase           " Smart case (if uppercase used, case sensitive)
set showmatch           " Show matching brackets
set cursorline          " Highlight current line
```

## Quick Reference Cheat Sheet

```
NAVIGATION          EDITING             SAVE/QUIT
h j k l - move      i - insert          :w - save
gg - start          a - append          :q - quit
G - end             o - new line below  :wq - save & quit
0 - line start      dd - delete line    :q! - quit no save
$ - line end        yy - copy line
w - next word       p - paste           SEARCH
b - prev word       u - undo            /text - search
                    Ctrl+r - redo       n - next match
                    . - repeat          :%s/a/b/g - replace
```

---

**Previous:** [05-basic-commands.md](05-basic-commands.md) | **Next:** [07-file-permissions.md](07-file-permissions.md)
