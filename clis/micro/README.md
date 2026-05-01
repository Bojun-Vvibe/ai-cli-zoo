# micro

## What it does
A **single-static-Go-binary terminal text editor with the desktop-
keyboard muscle-memory you already have** — Ctrl-S saves, Ctrl-C
copies, Ctrl-V pastes, Ctrl-Z undoes, Ctrl-F finds, Ctrl-Shift-up
extends the selection — over a true-color 256-color TUI that draws
fine on any modern `$TERM`, with multi-cursor editing (Ctrl-D adds
the next occurrence to the selection à la VS Code / Sublime),
mouse support including drag-to-select and click-to-place-cursor,
syntax highlighting for ~140 languages out of the box, split
horizontal / vertical panes (`Ctrl-E vsplit`), an integrated
terminal pane (`Ctrl-E term`) for running tests next to the code,
LSP via the third-party `comment` / `lsp` plugins, plugin manager
(`micro -plugin install lsp filemanager linter`) backed by Lua
plugin scripts, and JSON `~/.config/micro/settings.json` /
`~/.config/micro/bindings.json` config files that you can read
without learning a new DSL. The whole thing is ~12 MB, zero
runtime dependencies, drops onto a fresh Alpine box with one
`apk add micro` and onto a busybox container with one
`curl https://getmic.ro | bash`.

## Why it's interesting
Different shape from `vim` / `neovim` / [`helix`](../helix/) (modal
editors — powerful, but the *whole point* of micro is that Ctrl-S
saves the file, full stop, with no prior `:` and no prior `Esc`),
from `nano` (also modal-free and beginner-friendly but no
multi-cursor, no plugin system, no integrated terminal, and the
keybinds use the 1990s `^G ^O ^X` row instead of desktop Ctrl-S /
Ctrl-C / Ctrl-V), from `kakoune` (multi-cursor first but selection-
then-action modal grammar — micro is the *one-key-saves-and-the-
clipboard-is-the-system-clipboard* pick), from `emacs -nw` (deep,
extensible, but a 100 MB install plus an Elisp config and a `M-x`
muscle-memory tax), and from `code` / VS Code in WSL / `code-server`
(graphical or browser-based, requires a Node runtime and a websocket
to a server process — micro is the *SSH into the box, edit the
file, exit* shape with no server / runtime / extension host). Pick
micro when the actual ask is "I'm SSH'd into a teammate's debug
host, I have to edit one config file, I do not want to deliver a
30-minute vim tutorial first, and I want Ctrl-S to mean Save" — or
when onboarding a junior on a fresh container without a graphical
editor and you don't want them to learn modal editing on day one.
Do **not** pick it as a primary IDE on a 200k-LOC monorepo (use
neovim / helix / VS Code with real LSP setup), for `vimdiff`-class
3-way merges (use a real merge tool / [`mergiraf`](../mergiraf/)),
or when the editing happens inside `git rebase -i` / `git commit`
buffers and your `core.editor` is already vim (no reason to switch
the muscle).

## Niche category
Terminal text editor with desktop keybindings (Ctrl-S / Ctrl-C /
Ctrl-V), single static Go binary, multi-cursor, syntax-highlight,
plugin system — the "no-modal SSH editor" shape.

## Repo
https://github.com/zyedidia/micro

## Version pinned
`v2.0.15` (latest tagged release as of 2026-05-01)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`
  (https://github.com/zyedidia/micro/blob/master/LICENSE)

## Install
```sh
# Homebrew (macOS / Linux)
brew install micro

# Debian / Ubuntu (newer than apt; prefer the upstream installer or homebrew)
apt-get install micro

# Alpine
apk add micro

# One-line installer (drops a binary into the current directory)
curl https://getmic.ro | bash

# Go install
go install github.com/zyedidia/micro/v2/cmd/micro@latest
```

## Usage examples
```sh
# Open a file (Ctrl-S saves, Ctrl-Q quits, Ctrl-G shows the help pane)
micro path/to/file.go

# Open with a plugin loaded (LSP, linter, file-manager sidebar)
micro -plugin install lsp filemanager linter
micro path/to/project/

# Read from stdin into a scratch buffer (Ctrl-S to write it out)
kubectl get pod foo -o yaml | micro

# Set persistent options as JSON
cat > ~/.config/micro/settings.json <<'EOF'
{ "tabsize": 2, "tabstospaces": true, "colorscheme": "geany",
  "ruler": true, "softwrap": true, "autoindent": true }
EOF

# Inside the editor: Ctrl-E opens the command bar
#   > vsplit other.go         # open another file in a vertical split
#   > term                    # open an integrated terminal pane
#   > set colorscheme monokai # change theme on the fly
#   > tabswitch +1            # next tab

# Multi-cursor: Ctrl-D adds the next occurrence of the selection to a new cursor
```

## Date added
2026-05-01
