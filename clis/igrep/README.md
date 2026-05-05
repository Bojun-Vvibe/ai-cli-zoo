# igrep

> Snapshot date: 2026-05. Upstream: <https://github.com/konradsz/igrep>

**Interactive grep TUI built on top of ripgrep.** `igrep` (interactive grep)
wraps `ripgrep` in a full-screen terminal interface so you can run a query,
scroll a unified list of matches across files, preview surrounding context,
and jump directly into your `$EDITOR` at the matched line — without ever
copy-pasting a `path:line` triple. It's the missing "second pane" for the
ripgrep workflow: same regex engine, same speed, but the results become a
navigable thing instead of a wall of text the shell scrolled past.

## Repo + version + license

- Repo: <https://github.com/konradsz/igrep>
- Latest release: **`v1.3.0`** (2024-09-08)
- HEAD on `main`: `aa75630`
- License: **MIT** —
  <https://github.com/konradsz/igrep/blob/main/LICENSE>
- License path in repo: `LICENSE` (SPDX: `MIT`)
- Default branch: `main`
- Language: Rust (built on `ratatui` + the `grep` crate family that powers ripgrep)

## Install

```bash
# Homebrew
brew install igrep

# Cargo
cargo install igrep

# Arch
pacman -S igrep
```

```bash
# Search a regex across the current tree, then navigate matches in TUI
ig 'fn\s+handle_\w+' .

# Pass ripgrep flags through (e.g. only Rust files, hidden, follow symlinks)
ig -t rust --hidden --follow 'TODO\(.*\)'

# Open a specific match in $EDITOR (vim/nvim/helix/code) at the right line
# (Enter on a result; configure editor via --editor or $EDITOR)
ig --editor nvim 'unwrap\(\)' src/
```

## Niche

The "**ripgrep result browser**" slot. A normal `rg` invocation dumps all
matches to stdout — great for piping, terrible for "I want to look at each
hit and decide what to do." The historic answer was vim's `:grep` /
`quickfix`, fzf's `--preview` tricks (`rg ... | fzf --preview 'bat ...'`),
or one of the editor-native search panels (Helix's `space-/`, VS Code's
search view). `igrep` lives in the gap: a standalone TUI you can launch
from any shell, with no editor lock-in, that shares ripgrep's mental model
(globs, file-type filters, `--hidden`, `--follow`) one-to-one.

## Why it matters

- **Same regex engine as ripgrep** — `igrep` links the `grep-regex` and
  `grep-searcher` crates directly, so PCRE-ish syntax, multiline mode,
  word boundaries, and Unicode behave identically to `rg`. No surprise
  semantics gap when you alias `ig` next to `rg`.
- **Editor-agnostic jump-to-match** — works with any `$EDITOR` that takes
  `+LINE FILE` or `--goto FILE:LINE`. Vim, Neovim, Helix, Kakoune, VS
  Code (`code -g`), Zed, and Emacs (`emacsclient +line:col file`) are all
  supported via the `--editor` flag, so you don't need an editor plugin
  to get a productive search-and-fix loop.
- **Comparable CLIs** — for the "results-as-buffer" workflow:
  [`fzf`](../fzf/) + `rg --line-number | fzf --preview` is the DIY
  precursor, [`scooter`](../scooter/) and [`serpl`](../serpl/) cover the
  search-and-replace TUI niche, and [`grep-ast`](../grep-ast/) /
  [`ast-grep`](../ast-grep/) operate at the AST level instead of bytes.
  `igrep` is the simplest "rg, but interactive" option without committing
  to a refactor tool or an editor.
- **Tiny, single-binary, MIT** — under ~5 MB stripped, no daemon, no
  config required; reads `RIPGREP_CONFIG_PATH` so your existing `.ripgreprc`
  globs and ignores apply automatically.
