# fish-shell

> **A user-friendly POSIX-adjacent interactive shell with batteries
> included** — autosuggestions from your history as you type, syntax
> highlighting on the command line in real time, fish-script as a
> first-class scripting language (no `set -euo pipefail` dance),
> abbreviations that expand inline instead of opaque aliases, and
> man-page-derived completions for ~10 k commands out of the box.
> Pinned to **v4.6.0** (commit
> `6dedc3a41e8ddb80e711303bc6fd2d4022432c02`,
> [COPYING](https://github.com/fish-shell/fish-shell/blob/master/COPYING),
> GPL-2.0).

Source: <https://github.com/fish-shell/fish-shell>

## TL;DR

`fish` is what you reach for when you want zsh's interactive
ergonomics (history search, autosuggestions, completions) without
having to install `zsh-autosuggestions`, `zsh-syntax-highlighting`,
`fzf-tab`, and a 600-line `.zshrc` framework like Oh My Zsh first.
The defaults are the framework: start typing `git che` and a grey
suggestion of `git checkout main` (your last matching command)
appears inline — `→` accepts it. Commands turn red until they
resolve to a real binary, paths underline if they exist, quoted
strings light up. Tab completion is structured: hit `<Tab>` after
`git --` and you get a paged grid with descriptions parsed from the
git man pages, no plugin required. The cost is that fish-script is
**not POSIX**: `if [ -f foo ]; then ... fi` becomes `if test -f foo;
... end`, `$VAR` works but `${VAR:-default}` does not, function
syntax is `function foo; ...; end`. Most users solve this by keeping
fish as the *interactive* shell and writing portable scripts with a
`#!/bin/bash` shebang — `chsh -s $(which fish)` for the prompt,
`bash script.sh` for automation.

## Install

```bash
# Homebrew (macOS / Linux)
brew install fish

# Cargo (fish 4.x is Rust-based)
cargo install --locked fish

# Linux package managers
# Debian/Ubuntu: apt install fish
# Arch:          pacman -S fish
# Fedora:        dnf install fish
# Nix:           nix-env -iA nixpkgs.fish

# make it your login shell (after adding to /etc/shells)
echo "$(which fish)" | sudo tee -a /etc/shells
chsh -s "$(which fish)"

# verify
fish --version    # fish, version 4.6.0
```

## License

GPL-2.0 — see
[COPYING](https://github.com/fish-shell/fish-shell/blob/master/COPYING).
The shell binary is freely redistributable; GPL terms apply if you
ship a derivative that links fish-shell sources.

## One Concrete Example

```fish
# 1. interactive demo — paste into a fresh fish session
#    (the autosuggestion shows up in grey after the first run)
echo hello world
# ↑ press up-arrow → fuzzy-match recall on prefix
# →               → accept inline grey suggestion
# alt-→           → accept one word at a time

# 2. a real fish function in ~/.config/fish/functions/mkcd.fish
function mkcd --description 'mkdir + cd'
    mkdir -p $argv[1]; and cd $argv[1]
end

# 3. abbreviations expand inline (so `g co` becomes `git checkout`
#    in the buffer — not hidden behind an alias)
abbr -a g    git
abbr -a gco  git checkout
abbr -a gst  git status

# 4. universal variables — set once, persisted across all fish
#    sessions and survive reboots (no `export FOO=bar` in rc files)
set -Ux EDITOR nvim
set -Ux fish_greeting ""        # silence the welcome banner

# 5. completions are auto-generated from man pages
fish_update_completions          # rebuild from /usr/share/man/*
```

## Niche It Fills

**An interactive shell where the IDE-style ergonomics are the
default, not a plugin tax.** The space is bash / zsh / fish /
nushell / elvish / xonsh. Bash is universal but its interactive
surface is from 1989. Zsh is powerful but the moment you want
autosuggestions + syntax highlighting + nice completions you are
installing four plugins and a framework. Nushell is structured-data
first (every command outputs a table) — different niche. Fish sits
in the corner labelled *"a sane interactive shell out of the box,
at the cost of POSIX scripting compatibility."* Pick fish when you
spend most of your shell time typing commands interactively and you
are willing to write automation scripts in `bash` / `python` / `just`
instead of in your shell.

## Why use it

Three things `fish` does that pay off immediately:

1. **History-based autosuggestions in grey, inline.** You type
   `ssh`, fish shows `ssh user@host.example.com` in dim grey from
   your last matching command. `→` accepts the whole thing, `alt-→`
   accepts one word. No `Ctrl-R` modal popup, no fuzzy-finder
   sidecar — it just appears.
2. **Real-time syntax highlighting.** Unknown commands are red.
   Valid paths are underlined. Quoted strings are colored.
   Misbalanced parens jump out. You catch typos before pressing
   `Enter`.
3. **Completions parsed from man pages.** `git <Tab>` shows every
   subcommand with its one-line description, paged in a grid. No
   `compinit`, no `fpath` curation, no plugin to install. Run
   `fish_update_completions` once when you install a new tool and
   it learns the surface.

## Weakness

**Fish-script is not POSIX.** `[ -f foo ] && echo yes` does not
work. Bash one-liners copy-pasted from Stack Overflow may need
translation. Most users keep fish interactive-only and write
automation in bash with a `#!/bin/bash` shebang — at which point
the incompatibility is invisible. If your workflow is "paste long
bash snippets into the shell and tweak them," fish will frustrate
you and zsh is the better choice.

## When to choose

- You want zsh-style ergonomics without curating a plugin set.
- You write automation in `bash` / `python` / `just`, not in your
  interactive shell, so POSIX-incompat at the prompt is fine.
- You SSH into the same hosts repeatedly and want the
  autosuggestion to recall the full command.

## When NOT to choose

- Your `.bashrc` / `.zshrc` is 500 lines of POSIX glue you cannot
  rewrite — stay on bash/zsh.
- You need to run shell scripts from random GitHub READMEs that
  assume `bash` syntax in your interactive shell — use bash.
- You want structured-data piping (`ls | where size > 1mb`) — use
  [`nushell`](../nushell/) instead.
