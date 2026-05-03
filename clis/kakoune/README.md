# kakoune

> **Modal code editor inspired by Vim that inverts the
> "verb-then-object" grammar to "object-then-verb"** — every
> action begins by selecting (one or many ranges, with full
> multi-cursor as a first-class primitive), and the verb then
> operates on the selection. The result is interactive, visible
> editing (you see what you are about to change before you
> change it) and a UNIX-shaped architecture: each window is a
> separate client process, the editor server speaks to external
> tools via stdin/stdout, and there is no built-in scripting
> language — just shell. Pinned to **v2026.04.12**
> ([UNLICENSE](https://github.com/mawww/kakoune/blob/master/UNLICENSE),
> public domain).

Source: <https://github.com/mawww/kakoune>

## TL;DR

Where Vim says `d3w` ("delete three words" — verb, count,
object, blind), Kakoune says `3w d` (`w` extends the selection
by one word, `3w` by three, then `d` deletes what is highlighted
— object first, you see it, then verb). Multiple cursors are not
a plugin: `s` selects every regex match in the current
selection, `<a-s>` splits the selection on newlines into one
cursor per line, every subsequent keystroke applies to all
cursors at once. The editor is a daemon (`kak -d`); each
terminal window is a thin client (`kak -c session`); external
filters integrate with `:|sort`, `:!grep`, etc. — no embedded
Lua / VimScript / Python.

## Install

```bash
# Homebrew (macOS / Linux)
brew install kakoune

# Pre-built binaries / source
# https://github.com/mawww/kakoune/releases/tag/v2026.04.12
git clone --branch v2026.04.12 --depth 1 https://github.com/mawww/kakoune.git
cd kakoune/src && make && sudo make install

# Linux package managers
# Arch:   pacman -S kakoune
# Debian: apt install kakoune
# Fedora: dnf install kakoune
# Nix:    nix-env -iA nixpkgs.kakoune

# verify
kak -version    # v2026.04.12
```

First run drops you into a buffer with the default theme; user
config lives at `~/.config/kak/kakrc`. The plugin manager of
choice is [plug.kak](https://github.com/andreyorst/plug.kak) (no
plugin manager ships with the editor itself — same UNIX-y
philosophy).

## License

The Unlicense (public domain) — see
[UNLICENSE](https://github.com/mawww/kakoune/blob/master/UNLICENSE).
No restrictions; vendor and modify freely.

## One Concrete Example

```bash
# 1. Edit a file
kak src/main.rs

# Inside kak (object-first grammar):
# w        -> select to next word                (selection visible)
# d        -> delete the selection
# i / a    -> enter insert before / after the selection
# x        -> select the current line
# X        -> extend selection by one more line
# %        -> select the whole buffer
# /foo<ret>-> select first match of "foo" forward
# n        -> select next match (extending the search)
# s bar<ret>-> in current selection, select every match of "bar"
#             (this is multi-cursor — every subsequent edit applies
#              to all matches at once)
# <a-s>    -> split current selection on newlines (one cursor per line)
# c        -> delete selection AND enter insert mode (change)
# y / p    -> yank / paste
# u / U    -> undo / redo

# 2. Multi-cursor refactor: rename `foo` to `bar` in the visible
#    function, with edit-as-you-type confirmation
# %                  -> select whole buffer
# s\bfoo\b<ret>      -> one cursor on each whole-word match
# c bar <esc>        -> change all to "bar" simultaneously

# 3. Pipe a selection through an external filter (no plugin needed)
# %                  -> select whole buffer
# |jq .<ret>         -> replace selection with `jq .` of the selection
# %                  -> select whole buffer
# |sort -u<ret>      -> dedupe + sort

# 4. Daemon + clients: open the same session in two terminals
kak -d -s mywork                          # terminal A: start daemon
kak -c mywork src/lib.rs                  # terminal A: open client
kak -c mywork tests/lib.rs                # terminal B: second client
# Both clients see the same buffers, registers, and edit history.

# 5. Scripted edit from the shell (no need to open a UI)
echo 'edit foo.txt
%
| sort -u
write
quit' | kak -ui dummy
```

## Niche It Fills

**Modal editing redesigned around visible selections and shell
composability, instead of an embedded scripting language.** Vim
optimises for typed economy with a script-driven plugin
ecosystem; Kakoune optimises for *visible* edits (selection
first, then verb) and a UNIX-shaped extension surface (filters
in any language, communicated via stdin / stdout / the editor's
RPC socket). It is the realistic answer for someone who has the
modal-editing instinct but wants multi-cursor as a primitive and
does not want to learn VimScript / Lua to extend the editor.

## Why use it

1. **Selection-then-verb is teachable in an afternoon.** Every
   action shows you the highlighted target before you commit.
   The cost of a misremembered keystroke is a wrong highlight,
   not a wrong edit.
2. **Multi-cursor is a primitive, not a plugin.** `s<regex>` to
   select every match in the selection, `<a-s>` to split-on-
   newline, every subsequent keystroke applies to all cursors —
   the same workflow that requires `vim-multiple-cursors` or
   the (semi-broken) `:s/.../...` cycle in Vim.
3. **No embedded scripting language.** Extensions are shell
   scripts that pipe into `|`, `<a-|>`, `:!`, `:%sh{...}`, or
   the editor's JSON-RPC socket. The dependency surface of a
   Kakoune config is the dependency surface of `bash` plus
   whatever tools you call.
4. **Client / server is the default.** Each terminal window is a
   thin client (`kak -c <session>`); the editor itself is a
   single daemon. Open the same session in two terminals, both
   see the same buffers and undo history live.

## Vs Already Cataloged

- **Vs [`helix`](../helix/) / [`hx`](../helix/):** Helix is the
  "Kakoune-grammar editor with batteries (LSP, tree-sitter,
  themes, plugins) included." Kakoune is the "same grammar,
  bring-your-own-everything" version — smaller, older, stabler,
  but you wire LSP / formatters / linters yourself via the
  shell-extension surface (see `kak-lsp`). Helix for batteries
  included; Kakoune if you want the editor itself to stay
  small and deferred to UNIX tools.
- **Vs [`micro`](../micro/) / [`amp`](../amp/) (and Vim itself,
  not in catalog):** `micro` is the modeless,
  Ctrl-shortcut-style editor for users who do not want modal at
  all; `amp` is a different modal experiment in Rust. Kakoune
  occupies the "modal but rethought" spot and is the editor
  Helix's grammar is descended from.
- **Vs [`zellij`](../zellij/) / `tmux` (multiplexers):**
  orthogonal. The Kakoune client / server architecture means
  multiple terminal *clients* on one Kakoune *session* without
  needing a multiplexer at all — but pairing Kakoune with a
  multiplexer (one client per pane) is a common workflow.

## Caveats

- **Smaller ecosystem than Vim / Neovim / Helix.** Plugin count
  in the dozens, not thousands. The "I want a polished IDE-shaped
  experience out of the box" path goes through Helix; the "I
  want the world's biggest plugin ecosystem" path goes through
  Neovim.
- **No async LSP in core.** LSP support is provided by the
  external [`kak-lsp`](https://github.com/kakoune-lsp/kakoune-lsp)
  bridge — works well, but is one more service to install,
  configure, and update independently from the editor.
- **Windows is unsupported.** Linux, macOS, and BSDs only;
  Windows users run it under WSL.
- **Documentation is reference-shaped, not tutorial-shaped.**
  `:doc` inside the editor is excellent as a reference, but the
  smoothest learning path is the community-written
  [Kakoune for the Vim editor](https://github.com/mawww/kakoune/wiki)
  pages plus reading other people's `kakrc` files.
- **Daemon model surprises some users.** `kak -clear` to wipe
  stale sessions, `:kill` to take the daemon down, and a
  crashed daemon takes every client with it; treat the session
  the way you treat a `tmux` server.
