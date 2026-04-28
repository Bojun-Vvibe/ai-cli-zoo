# xplr

- **Repo:** https://github.com/sayanarijit/xplr
- **Version:** v1.1.0 (latest tagged release, 2025-12)
- **License:** MIT ([LICENSE](https://github.com/sayanarijit/xplr/blob/main/LICENSE))
- **Language:** Rust (with Lua scripting layer)
- **Install:** `brew install xplr` · `cargo install --locked xplr` · `nix-env -iA nixpkgs.xplr` · prebuilt binaries on the GitHub releases page

## What it does

`xplr` is a hackable, minimal, fast TUI file explorer that treats the
file manager itself as a thin engine and pushes nearly all behaviour
into a Lua configuration layer. The C-fast Rust core renders a
two/three-pane terminal UI (current directory listing, optional parent
preview, optional contents preview), watches the filesystem for live
updates, and dispatches every keystroke through a programmable mode
machine; everything you see on screen — the layout, the status line,
the sort order, the filter rules, the colour scheme, the file-type
icons, the keybindings, the "what happens when I press Enter on a
.tar.gz" — is decided by Lua. It ships with a sensible default config
so it works out of the box (j/k to move, h/l to ascend/descend, Space
to toggle a selection, Enter to enter, q to quit), but the killer
feature is `--print-pwd-as-result` / `--read-only` / `--on-load`
mode-piping: `xplr` is designed to be invoked as a subprocess of your
shell, your editor, your git workflow, or your build script, with the
selected paths printed to stdout for the parent process to consume.
That makes it less a "file manager" in the Midnight Commander sense
and more a re-usable TUI filesystem picker that any pipeline can plug
into. A growing ecosystem of community plugins (fzf integration, trash
bin, git status overlay, image preview via chafa/kitty graphics, batch
rename via `$EDITOR`, archive in/out, dual-pane mode, search via fd,
diff via delta) covers most of what monolithic file managers ship in
the box, but each one is opt-in Lua you read and audit before loading.

## When to pick it / when not to

Pick `xplr` when you want a file manager that is *programmable the way
your editor is programmable* — you write Lua, you bind keys, you
compose new modes, and the binary stays out of your way. The "invoke
as a subprocess and read the selected paths back" model is the unique
value: a shell function `cdx() { cd "$(xplr --print-pwd-as-result)"; }`
gives you a Norton-style picker for `cd`; a vim/neovim binding `:!xplr`
gives you a file picker that doesn't depend on a plugin ecosystem; a
script can `mapfile -t files < <(xplr)` to let the user multi-select
inputs to a batch operation. Compared to the peers: pick `xplr` over
[`yazi`](../yazi/) when you want extreme hackability and a mature Lua
plugin surface and don't mind writing config (yazi is faster to adopt
out of the box and has built-in async preview); over [`broot`](../broot/)
when you want a classic two-pane pager rather than broot's tree-with-
fuzzy-search model; over [`nnn`](https://github.com/jarun/nnn) when you
prefer Lua to nnn's plugin-shell-script convention; over
[`lf`](https://github.com/gokcehan/lf) when you've outgrown lf's
single-file config and want a real scripting language; and over
[`tere`](../tere/) when you want a full file manager rather than tere's
"type-to-cd, then exit" minimalism.

Skip it when you want zero configuration and "just works" defaults —
xplr's defaults are usable but the project is *built around* the
expectation that you'll write Lua, and most of the cool screenshots in
the wild are someone's customised setup. Skip it when you need
production-grade async previews of huge files (large videos, multi-GB
log files) — preview is plugin-driven and quality varies; yazi has a
more polished story here. Skip it when your colleagues need to share
the workflow on a stock terminal — every machine needs xplr installed
*and* the Lua config synced. And skip it for pure
"navigate-and-cd-then-exit" use cases where [`tere`](../tere/) or
[`zoxide`](../zoxide/) is a tenth of the install footprint and 100% of
the value.

## Example invocations

```bash
# Just open the explorer in the current directory
xplr

# Open it in a specific directory
xplr ~/Projects

# Pick a directory and cd to it (shell function)
cdx() {
  local dir
  dir=$(xplr --print-pwd-as-result) || return
  [ -n "$dir" ] && cd -- "$dir"
}

# Multi-select files and feed them to a batch command
xplr | xargs -d '\n' -I{} cp -v {} /tmp/staged/

# Read-only mode — useful as a $PAGER-style file inspector
xplr --read-only ~/var/log

# As a neovim file picker
:!xplr --print-pwd-as-result

# Drop into xplr from inside a git repo to pick changed files for staging
xplr --on-load 'BashExec ["git status -s | awk \"{print \\$2}\""]'

# Use a project-local config (per-repo modes / keybinds)
xplr --extra-config ./.xplr.lua
```
