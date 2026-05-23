# Neovim Cheatsheet

## Mental Model

Neovim is a **modal editor** — the keyboard behaves differently depending on which mode you're in. This is the single most important concept. Modes exist so every key on the keyboard becomes a command without needing modifier keys. The learning curve is front-loaded: once muscle memory kicks in, editing becomes significantly faster than any GUI editor.

**Modes at a glance:**

| Mode | How to enter | What it does |
|---|---|---|
| Normal | `Esc` / `Ctrl+c` | Navigate and run commands — where you spend most time |
| Insert | `i`, `a`, `o` | Type text |
| Visual | `v`, `V`, `Ctrl+v` | Select text |
| Command | `:` | Run Ex commands (save, quit, search, etc.) |
| Replace | `R` | Overwrite text |

---

## Install & Minimal Setup

```bash
# Linux (apt)
sudo apt install neovim

# macOS
brew install neovim

# Latest stable release (recommended)
# https://github.com/neovim/neovim/releases

# Config location
~/.config/nvim/init.lua   # Lua config (modern — use this)
~/.config/nvim/init.vim   # VimScript config (legacy)

# Check health
:checkhealth
```

---

## Core Concepts

### 1. The Grammar of Normal Mode
Commands follow a pattern: `[count] [operator] [motion]`

```
d w     → delete a word
d 3 w   → delete 3 words
c i "   → change inside quotes
y a p   → yank (copy) a paragraph
> >     → indent current line
```

Operators: `d` (delete), `c` (change), `y` (yank), `>` / `<` (indent), `=` (format), `g~` (toggle case)

### 2. Motions (Navigate Without Arrow Keys)

```
h j k l     → ← ↓ ↑ →  (basic movement)
w / b       → next / previous word start
e           → end of word
0 / ^       → start of line (0 = column 0, ^ = first non-blank)
$           → end of line
gg / G      → top / bottom of file
5G          → go to line 5
Ctrl+d / u  → scroll half page down / up
Ctrl+f / b  → scroll full page down / up
%           → jump to matching bracket
*           → search for word under cursor (forward)
#           → search for word under cursor (backward)
f{char}     → jump to next occurrence of {char} on current line
t{char}     → jump to just before next {char}
; / ,       → repeat f/t forward / backward
```

### 3. Insert Mode Entry Points

```
i   → insert before cursor
a   → insert after cursor
I   → insert at start of line
A   → insert at end of line
o   → open new line below and insert
O   → open new line above and insert
s   → delete character and insert
S   → delete line and insert
c{motion} → delete motion and insert (e.g. cw = change word)
```

### 4. Visual Mode

```
v           → character-wise visual
V           → line-wise visual
Ctrl+v      → block visual (column selection)

# After selecting:
d           → delete selection
y           → yank (copy)
c           → change (delete and insert)
>  <        → indent / dedent
~           → toggle case
:           → run command on selection
```

### 5. Editing Commands (Normal Mode)

```
x           → delete character under cursor
dd          → delete current line
D           → delete to end of line
yy          → yank current line
p / P       → paste after / before cursor
u           → undo
Ctrl+r      → redo
.           → repeat last change (extremely useful)
r{char}     → replace character under cursor with {char}
~           → toggle case of character under cursor
>>  <<      → indent / dedent line
==          → auto-indent line
J           → join line below to current line
```

### 6. Search & Replace

```
/pattern    → search forward
?pattern    → search backward
n / N       → next / previous match
:%s/old/new/g   → replace all in file
:%s/old/new/gc  → replace all with confirmation
:5,10s/old/new/g → replace in lines 5–10

# In pattern: \c = case insensitive, \C = case sensitive
/\cpattern
```

### 7. Buffers, Windows, Tabs

```
# Buffers (open files)
:e filename     → open file in current buffer
:ls             → list open buffers
:b2             → switch to buffer 2
:bd             → delete (close) current buffer
:bn / :bp       → next / previous buffer

# Windows (splits)
:sp filename    → horizontal split
:vsp filename   → vertical split
Ctrl+w h/j/k/l  → navigate between windows
Ctrl+w =        → equalize window sizes
Ctrl+w q        → close window

# Tabs
:tabnew         → new tab
gt / gT         → next / previous tab
```

### 8. File Operations

```
:w              → save
:w filename     → save as
:q              → quit
:q!             → quit without saving
:wq / ZZ        → save and quit
:qa             → quit all windows
:e!             → reload file from disk
```

### 9. Marks & Jumps

```
ma              → set mark 'a' at cursor position
`a              → jump to mark 'a' (exact position)
'a              → jump to line of mark 'a'
``              → jump back to previous position
Ctrl+o / Ctrl+i → jump list backward / forward
```

---

## Most-Used Patterns

### Text Objects (Combine with operators)

```
i = "inside"   a = "around" (includes surrounding chars)

ciw     → change inside word
ci"     → change inside double quotes
ca"     → change around double quotes (includes the quotes)
di(     → delete inside parentheses
yi{     → yank inside curly braces
vap     → visually select a paragraph
das     → delete a sentence
```

### Macros

```
qa      → start recording macro into register 'a'
q       → stop recording
@a      → play macro 'a'
10@a    → play macro 'a' 10 times
@@      → replay last macro
```

### Global Command

```
:g/pattern/d        → delete all lines matching pattern
:g/pattern/normal dd → run normal mode command on matching lines
:v/pattern/d        → delete all lines NOT matching pattern
```

---

## Lua Config Basics (`~/.config/nvim/init.lua`)

```lua
-- Options
vim.opt.number = true           -- show line numbers
vim.opt.relativenumber = true   -- relative line numbers (great for motions)
vim.opt.tabstop = 2
vim.opt.shiftwidth = 2
vim.opt.expandtab = true        -- spaces instead of tabs
vim.opt.wrap = false
vim.opt.ignorecase = true
vim.opt.smartcase = true        -- case-sensitive if uppercase in pattern
vim.opt.termguicolors = true
vim.opt.scrolloff = 8           -- keep 8 lines visible around cursor

-- Leader key
vim.g.mapleader = " "           -- space as leader

-- Keymaps: vim.keymap.set(mode, lhs, rhs, opts)
vim.keymap.set("n", "<leader>w", ":w<CR>",        { desc = "Save file" })
vim.keymap.set("n", "<leader>q", ":q<CR>",        { desc = "Quit" })
vim.keymap.set("n", "<Esc>",     ":nohlsearch<CR>", { desc = "Clear search highlight" })
vim.keymap.set("i", "jk",        "<Esc>",          { desc = "Exit insert mode" })

-- Move lines in visual mode
vim.keymap.set("v", "J", ":m '>+1<CR>gv=gv")
vim.keymap.set("v", "K", ":m '<-2<CR>gv=gv")
```

### Plugin Manager — lazy.nvim

```lua
-- Bootstrap lazy.nvim (paste at top of init.lua)
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
  vim.fn.system({ "git", "clone", "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git", lazypath })
end
vim.opt.rtp:prepend(lazypath)

require("lazy").setup({
  -- File explorer
  { "nvim-tree/nvim-tree.lua" },

  -- Fuzzy finder
  { "nvim-telescope/telescope.nvim", dependencies = { "nvim-lua/plenary.nvim" } },

  -- Syntax highlighting
  { "nvim-treesitter/nvim-treesitter", build = ":TSUpdate" },

  -- LSP
  { "neovim/nvim-lspconfig" },
  { "williamboman/mason.nvim" },          -- LSP installer

  -- Autocompletion
  { "hrsh7th/nvim-cmp" },
  { "hrsh7th/cmp-nvim-lsp" },

  -- Theme
  { "catppuccin/nvim", name = "catppuccin" },
})
```

---

## Gotchas

- **Esc to Normal first** — everything starts from Normal mode. If something isn't working, press `Esc` first.
- **`:` commands are not case-sensitive** — `:W` won't save; Vim commands are lowercase.
- **`d` puts text in a register** — deleting with `d` overwrites the default register. To truly delete without affecting clipboard, use `"_d` (black hole register).
- **Registers** — `"a y` yanks into register `a`; `"a p` pastes from it. `"+y` yanks to system clipboard.
- **`cw` vs `ciw`** — `cw` changes from cursor to end of word; `ciw` changes the whole word regardless of cursor position.
- **Swapfiles** — Neovim creates `.swp` files when editing. If you see a swap warning, press `D` to delete it if there's no unsaved work.

---

## Quick Links

- [Neovim Docs](https://neovim.io/doc/)
- [`:help`](https://neovim.io/doc/user/) — built-in help is comprehensive; `:help motion` etc.
- [Kickstart.nvim](https://github.com/nvim-lua/kickstart.nvim) — minimal opinionated starter config
- [LazyVim](https://lazyvim.org) — full distribution built on lazy.nvim
- [Vim Adventures](https://vim-adventures.com) — learn motions as a game
- [Practical Vim (book)](https://pragprog.com/titles/dnvim2/practical-vim-second-edition/) — best book on the subject
