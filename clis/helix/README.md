# helix

> **A post-modern modal text editor with built-in LSP, tree-sitter
> syntax, and multiple selections** — a single Rust binary
> (`hx`) that ships with language-server config for ~150
> languages out of the box, no plugin install needed. Pinned to
> **25.07.1**
> ([LICENSE](https://github.com/helix-editor/helix/blob/master/LICENSE),
> MPL-2.0).

Source: <https://github.com/helix-editor/helix>

## TL;DR

`helix` (`hx` on the command line) is a Kakoune-inspired,
Vim-adjacent modal editor that bakes in everything Vim users
spend a weekend bolting on: a real LSP client, tree-sitter
syntax + structural selections, multiple cursors as the
primary editing primitive, fuzzy-find file picker, and live
grep — all in one ~30 MB static binary. No `init.vim`, no
plugin manager, no Python runtime; `hx file.go` opens a file
with `gopls` already wired up.

## Install

```bash
# Homebrew (macOS / Linux)
brew install helix

# Linux package managers
# Arch:   pacman -S helix
# Nix:    nix-env -iA nixpkgs.helix

# Cargo (any OS with a Rust toolchain)
cargo install --git https://github.com/helix-editor/helix helix-term --locked

# verify
hx --version    # helix 25.07.1

# launch
hx              # open the picker in cwd
hx path/to/file
```

## License

MPL-2.0 — see
[LICENSE](https://github.com/helix-editor/helix/blob/master/LICENSE).
File-level copyleft: modifications to MPL-licensed source files
must be shared, but you can link MPL code into a larger
proprietary work without infecting the whole project.

## Niche It Fills

**Modal editing without a plugin weekend.** Vim and Neovim are
endlessly customisable, but the price is that "good defaults"
are a personal config you carry between machines for years.
Helix's pitch: ship the modern editor everyone ends up
configuring Neovim into, as the default install. LSP, syntax,
multi-cursor, file picker, grep — all on first launch.

## Why it pairs with coding agents

For agent workflows that emit diffs or whole-file rewrites,
Helix's tree-sitter selections (`mif` = "select inside
function", `mim` = "select inside match pair") let you scope
the agent's context to exactly one function or one block in
two keystrokes — much cheaper than packing the whole file.
Multi-cursor + LSP rename means manual touch-ups after an
agent edit are usually one selection plus one keystroke.

## Vs Already Cataloged

- **Vs editor-embedded agents like [`avante.nvim`](../avante.nvim/),
  [`cline`](../cline/), [`codecompanion.nvim`](../codecompanion.nvim/):**
  Helix is the editor itself, not an agent. Pair it with any
  CLI agent in this catalog ([`opencode`](../opencode/),
  [`aider`](../aider/), [`crush`](../crush/)) — they edit files
  on disk, Helix renders the diff. There's no first-party
  Helix-embedded agent yet (community plugin work is ongoing
  upstream).
- **Vs `vim` / `nvim`:** Helix's selection-then-action model
  (`mi(` then `d`) flips Vim's verb-noun (`di(`); cleaner for
  some, jarring for others. Helix has no plugin system in the
  upstream release line — strict yes-or-no on the curated
  feature set.

## Caveats

- **No plugin system in the stable release.** A Steel-Scheme
  plugin runtime is in long-running development on `master` but
  not yet shipped in a tagged release. If your editing flow
  depends on a Vim plugin ecosystem, that's a hard blocker
  today.
- **Selection-first model takes a week.** Coming from Vim, the
  inverted verb order is a real adjustment; tutorials inside
  `hx` (`:tutor`) are mandatory.
