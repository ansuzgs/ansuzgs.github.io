---
title: "My Terminal Setup: Neovim 0.12, tmux, and Going Native"
date: 2026-05-27
draft: false
tags: ["neovim", "tmux", "dotfiles", "tooling"]
description: "A minimalist terminal development environment built on Neovim 0.12's native APIs — no plugin managers, no completion frameworks, no abstractions I don't control."
---

Every developer has opinions about their editor. Most of those opinions are wrong. Mine are also probably wrong, but at least I understand every line in my config.

This post walks through the terminal setup I use for C development, note-taking, and everything in between. The [dotfiles are on Codeberg](https://codeberg.org/Pablogs/dotfiles) if you want to skip the words and read the code. But if you're curious about *why* things are the way they are, why I don't use Mason, why there's no `nvim-cmp`, why my shell config is 130 lines instead of a framework, keep reading.

## The Stack

The whole thing fits in a sentence: Alacritty as the terminal, Neovim 0.12 with native LSP and completion, tmux for session management, zsh without frameworks, and two bash scripts that tie it together.

That's it. No Starship. No Oh My Zsh. No lazy.nvim. The plugin count in Neovim is 8. The `.zshrc` is a single file. The tmux config has zero dependencies beyond tmux itself (plus TPM for session persistence, because I got tired of losing my layout after reboots). Alacritty is configured in a single TOML file, font, colors, done.

The philosophy is simple: I want to understand every line. If I can't explain what a line does and why it's there, it doesn't belong.

## Alacritty: The Terminal That Does Nothing

Alacritty is a GPU-accelerated terminal emulator. It's fast. It has no tabs, no splits, no dropdown mode, no built-in multiplexer. It renders text and gets out of the way.

That's exactly what I want. tmux handles sessions and panes. Neovim handles editing. The terminal's job is to render glyphs quickly and not interfere. Alacritty does that better than anything else I've tried.

The config is ~60 lines of TOML:

```toml
[window]
padding = { x = 4, y = 4 }
decorations = "None"
startup_mode = "Maximized"

[font]
size = 12.0

[font.normal]
family = "JetBrainsMono Nerd Font"
style = "Regular"

[cursor.style]
shape = "Block"
blinking = "Off"

[terminal]
osc52 = "CopyPaste"
```

The colors are Kanagawa Wave, the same theme running in Neovim and tmux's status bar. Everything looks like one application. The `osc52 = "CopyPaste"` line enables clipboard passthrough so yanking in Neovim inside tmux inside Alacritty actually reaches the system clipboard. Without it, you're in clipboard purgatory.

JetBrainsMono Nerd Font because it's the most legible monospace font I've found for code, and the Nerd Font variant includes the glyphs that gitsigns and other tools use. No transparency, no blur, no background image, legibility over aesthetics, always.

## Neovim 0.12: The Year of Going Native

Neovim 0.12 changed the game for anyone who was already suspicious of the plugin stack. Two features made it possible to delete a lot of dependencies:

**`vim.pack`**, a built-in package manager. You declare plugins with `vim.pack.add()`, and Neovim handles cloning, updating, and loading. It asks you to confirm on first launch. No bootstrap snippet, no `lazy.nvim` download dance. It does less than lazy.nvim, no lazy-loading, no priority ordering, no build steps, but for 8 plugins, I don't need any of that.

```lua
vim.pack.add({
  { src = 'https://github.com/nvim-treesitter/nvim-treesitter', version = 'main' },
  { src = 'https://github.com/neovim/nvim-lspconfig' },
  { src = 'https://github.com/nvim-lua/plenary.nvim' },
  { src = 'https://github.com/nvim-telescope/telescope.nvim' },
  { src = 'https://github.com/nvim-telescope/telescope-fzf-native.nvim', version = 'main' },
  { src = 'https://github.com/lewis6991/gitsigns.nvim' },
  { src = 'https://github.com/christoomey/vim-tmux-navigator' },
  { src = 'https://github.com/rebelot/kanagawa.nvim' },
})
```

Eight plugins. Treesitter for parsing, lspconfig for server definitions, Telescope for finding things, gitsigns for the gutter, vim-tmux-navigator for seamless pane movement, and a colorscheme. That's the whole list.

**`vim.lsp.config()` + `vim.lsp.enable()`**, native LSP configuration. In older Neovim, you needed `require('lspconfig').clangd.setup{}` to configure a language server. Now you call `vim.lsp.config('clangd', { ... })` and `vim.lsp.enable('clangd')` directly. lspconfig is still useful, it provides the *definitions* of servers (filetypes, root markers, default commands), but the configuration API is Neovim's own.

```lua
vim.lsp.config('clangd', {
  cmd = {
    'clangd',
    '--background-index',
    '--clang-tidy',
    '--completion-style=detailed',
    '--header-insertion=never',
    '-j=4',
  },
  filetypes    = { 'c', 'cpp' },
  root_markers = { 'compile_commands.json', '.clangd', 'Makefile', 'Kbuild', '.git' },
})

vim.lsp.enable('clangd')
```

**`vim.lsp.completion`**, built-in completion. This is the big one. For years, the Neovim completion story was: install nvim-cmp, plus cmp-nvim-lsp, plus cmp-buffer, plus cmp-path, plus a snippet engine, plus a snippet source. Five or six plugins just to get an autocompletion menu. Now it's one function call:

```lua
vim.lsp.completion.enable(true, client.id, buf, { autotrigger = true })
```

The completion menu appears automatically when you type a trigger character (`.`, `->`, `::`). `C-n`/`C-p` navigate, `C-y` accepts, `C-Space` forces the menu open. These are the same keybindings Vim has used since 1991. No plugin needed.

Is it as feature-rich as nvim-cmp or blink.cmp? No. There's no fuzzy matching on completion items, no ghost text, no automatic import insertion. I don't need those things. For C development, where completions are struct fields, function names, and macros, the native menu does the job. The items come from clangd, which already does the heavy lifting.

This is a hard requirement: Neovim must be 0.12 or later. The install script checks the version and refuses to continue if it's older. `vim.pack`, `vim.lsp.config`, and `vim.lsp.completion` don't exist in 0.11, there's no graceful degradation, it just won't work.

### The Tradeoff

Let me be honest about the tradeoff. Going native means giving up:

- **Mason**: I install LSP servers manually (`pacman -S clangd lua-language-server marksman`). If I switch to a new machine, I run the install script. It's slightly more friction than `:MasonInstall clangd`, but it means I know exactly what version is running and where the binary lives.
- **Lazy loading**: All 8 plugins load at startup. Total startup time is around 60-80ms. I can live with that.
- **Rich completion UI**: No documentation popup next to the completion item, no source labels, no fancy formatting. The native popup with `completeopt = { 'menu', 'menuone', 'noselect', 'popup' }` shows the docs in a floating window. It's enough.

The gain: my entire Neovim config is ~400 lines of Lua across 6 files. I understand all of it. When something breaks, I know where to look. When Neovim releases 0.13, I won't be waiting for five plugins to update before I can upgrade.

## tmux: Sessions as Workspaces

My tmux config is a single file. The only external dependency is TPM for tmux-resurrect and tmux-continuum, session persistence across reboots. The interesting part isn't the config itself (prefix `C-a`, vi keys, splits with `|` and `-`, standard stuff). The interesting part is the sessionizer.

### tmux-sessionizer

Stolen from ThePrimeagen and simplified. You press `C-f` from anywhere, inside Neovim, in a shell, doesn't matter, and a fuzzy finder (fzf) shows all your project directories. Pick one, and you're in a tmux session named after that project, `cd`'d into that directory. If the session already exists, it switches to it instead of creating a new one.

```bash
selected=$(find "${SEARCH_DIRS[@]}" -mindepth 1 -maxdepth 1 -type d 2>/dev/null \
    | sort \
    | fzf \
        --prompt="project > " \
        --height=75% \
        --reverse \
        --preview='ls -la --color=always {} | head -20; echo; \
                   git -C {} log --oneline -5 2>/dev/null || echo "(no git)"')
```

The preview window shows the directory listing and the last 5 git commits, so you can tell at a glance which project is which before selecting. There's also a `--switch` mode (`C-a s`) that lists active sessions with a live preview of the pane contents, useful when you have four or five projects open and can't remember which window has your build output.

The kernel coding style (tabs=8, colorcolumn=80) is handled inside Neovim by an autocmd that detects `Kbuild`/`Kconfig` in the project tree. No separate config, no `NVIM_APPNAME` tricks, one config adapts to the project it's editing.

### Session Persistence

For months I ran tmux without tmux-resurrect. Every reboot meant recreating my session layout. For a while I told myself this was fine, "it's minimalist." It wasn't minimalist. It was annoying.

tmux-resurrect saves your session layout (windows, panes, working directories) and restores it on the next tmux start. tmux-continuum automates the saves every 15 minutes. Together they cost about 15 lines in `.tmux.conf` and zero cognitive overhead. This is the one case where I accepted a plugin manager (TPM) because the alternative, manually serializing tmux state, is a yak-shave I don't want.

## Zsh: The Shell Without a Framework

I switched from bash to zsh, but not to Oh My Zsh. Oh My Zsh is a framework, it loads hundreds of files, ships with dozens of plugins you don't need, and adds measurable startup latency. I want my shell to start in under 50ms.

My `.zshrc` is one file. It does:

- **History sharing** across sessions (the single best reason to use zsh over bash)
- **Completion** with the built-in `compinit`, case-insensitive, menu-based, with colors
- **Vi mode** with `bindkey -v` (because muscle memory from Neovim)
- **A minimal prompt** that shows the directory and git branch. No Starship, no Powerlevel10k, just `vcs_info` and `PROMPT_SUBST`
- **Two plugins**: `zsh-autosuggestions` (fish-style command suggestions from history) and `zsh-syntax-highlighting` (colors commands red/green based on validity). Both clone themselves on first shell start. No plugin manager.

```zsh
_ensure_plugin() {
    local repo="$1"
    local name="${repo##*/}"
    local dir="$ZSH_PLUGIN_DIR/$name"
    [[ ! -d "$dir" ]] && git clone --depth=1 "https://github.com/$repo.git" "$dir"
    for init in "$dir/$name.zsh" "$dir/$name.plugin.zsh"; do
        [[ -f "$init" ]] && source "$init" && return
    done
}

_ensure_plugin "zsh-users/zsh-autosuggestions"
_ensure_plugin "zsh-users/zsh-syntax-highlighting"
```

Seven lines of shell replace an entire plugin manager. The plugins clone on first run and source on every run. To update: `git pull` in each directory. I wrapped that in a `zsh-update-plugins` function for convenience.

### Why Zsh Over Bash

Honest answer: `SHARE_HISTORY`. Being able to type a command in one terminal and immediately search for it in another terminal's history is worth the switch alone. The rest, better completion, glob qualifiers, parameter expansion flags, is nice but not essential. If bash ever gets proper history sharing, I'd consider switching back. It won't, so I won't.

## Notes: A System Without Obsidian

I take notes in plain markdown files managed by two tools: `nz` (a bash CLI) and a `notes.lua` module in Neovim.

From the terminal, `nz` creates zettelkasten notes, daily journal entries, weekly reviews, and inbox captures. From Neovim, `<Space>nn` searches notes by filename, `<Space>ng` greps inside them, and `<Space>nd` opens today's journal. Everything lives in `~/notes` with a flat structure:

```
notes/
├── zk/           # permanent notes (zettelkasten)
├── journal/      # daily logs, weekly reviews
├── inbox/        # quick captures, unsorted
└── projects/     # per-project notes
```

The notes use YAML frontmatter with a `tags: []` field. `nz -t` extracts all tags across all notes and lets you filter by tag with fzf. Wiki-style `[[links]]` connect notes, and `<Space>nl` inserts a link by picking from a Telescope search.

Is this as powerful as Obsidian? No. There's no graph view, no backlink panel, no Dataview queries. But every note is a plain text file synced via Nextcloud, editable in any text editor, greppable with `rg`, and version-controlled with git. When Obsidian pivots, or gets acquired, or decides my notes should live in their cloud, my files are still just files.

## Stow: How the Dotfiles Get Where They Need to Go

The whole repo is structured for [GNU Stow](https://www.gnu.org/software/stow/). Each top-level directory is a "package" whose internal structure mirrors the path from `$HOME`:

```
nvim/.config/nvim/init.lua          → ~/.config/nvim/init.lua
tmux/.tmux.conf                     → ~/.tmux.conf
alacritty/.config/alacritty/...     → ~/.config/alacritty/...
zsh/.zshrc                          → ~/.zshrc
scripts/.local/bin/nz               → ~/.local/bin/nz
scripts/.local/bin/tmux-sessionizer → ~/.local/bin/tmux-sessionizer
```

`stow -t ~ nvim` creates a symlink from `~/.config/nvim` pointing into the repo. Edit the file in the repo, and the change is live. No copying, no syncing, no rsync scripts. `stow -D nvim` removes the symlink cleanly. This is the best approach I've found for managing dotfiles, it's been around since 1993 and it still works perfectly.

## The Install Script

There's an `install.sh` that handles everything: installs system packages (Arch, Fedora, and Debian/Ubuntu), verifies that Neovim is 0.12 or later, runs `stow` for each package, downloads LSP server binaries, installs TPM, and optionally switches your default shell to zsh. It's idempotent, run it twice and nothing breaks.

```bash
git clone https://codeberg.org/Pablogs/dotfiles.git ~/.dotfiles
cd ~/.dotfiles
./install.sh
```

The Neovim version check is a hard gate. If the installed version is newer than 0.12, the script prints an error and exits. There's no point linking a config that will immediately fail to load.

For people who don't trust install scripts (reasonable), there's a `--link-only` flag that just creates the symlinks without touching your package manager.

## What's Missing

This is a work in progress. Things I know I need to add:

- **`.editorconfig`** at the repo root — so the dotfiles themselves follow consistent formatting
- **`.clang-format`** for C userspace projects — my kernel work uses `checkpatch.pl`, but standalone C projects need something
- **`.gitignore` global**, to stop `*.o`, `.DS_Store`, and vim swap files from showing up in every repo

I'll update this post when those land. Or I'll write a new one. Whichever happens first.

## The Point

None of this is novel. People have been configuring terminal editors since before I was born. But the specific combination, Alacritty as a dumb-fast terminal, Neovim 0.12's native APIs eliminating the need for completion plugins and LSP managers, a single config that adapts to kernel and userspace projects via autocmds, a note system that runs on `grep` and `fzf`, is worth documenting, if only so I remember how to set it up on my next machine.

The repo is at [codeberg.org/Pablogs/dotfiles](https://codeberg.org/Pablogs/dotfiles). Use whatever's useful. Ignore the rest.

*Full source code on [Codeberg](https://codeberg.org/Pablogs/dotfiles) · Runs on Arch Linux with Alacritty, Neovim 0.12, tmux, zsh*
