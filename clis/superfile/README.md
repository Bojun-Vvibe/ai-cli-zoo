# superfile

> **A modern, opinionated TUI file manager
> with multi-pane navigation, image / video /
> PDF preview, sidebar pinning, and a
> configurable hotkey + theme system** — one
> Go binary (`spf`) that turns the terminal
> into a two-column file browser with the
> ergonomics of a desktop file app. Pinned to
> **v1.5.0**
> ([LICENSE](https://github.com/yorukot/superfile/blob/main/LICENSE),
> MIT).

Source: <https://github.com/yorukot/superfile>

## TL;DR

`superfile` (binary `spf`) is a Bubble Tea–
based file manager that shows **multiple file
panels side by side**, an always-visible
sidebar with pinned + default directories
(Home, Downloads, Documents, Trash on Linux),
and a process bar at the bottom for ongoing
copy / move / extract operations. The 1.5.0
line added **video and PDF preview** (renders
the first frame / page as an inline image via
the terminal's image protocol), **multi-column
file panel views** (date / size / permission
columns), **per-extension `open_with` editor
mapping** (open `.go` in `nvim`, `.png` in
`feh`), **fast configurable jump navigation**,
and **terminal stdout passthrough for shell
commands** so scripted file operations stream
their output back into the panel without
breaking the TUI. Configuration is plain TOML
under `~/.config/superfile/` (hotkeys, theme,
default editors, pinned dirs); themes follow
the same scheme as Helix / Zellij so existing
palettes (Catppuccin, Tokyonight, Everforest)
drop in. Launch supports both directory and
file targets — `spf /a/b/c.txt` opens `/a/b`
with `c.txt` selected — which makes it usable
as the destination of `xdg-open`-style file-
manager intents from other tools.

## Install

```bash
# Homebrew
brew install superfile

# Go (from source)
go install github.com/yorukot/superfile@latest

# Arch Linux (AUR)
yay -S superfile-bin

# prebuilt binaries (Linux / macOS / Windows, amd64 + arm64)
# https://github.com/yorukot/superfile/releases

# verify
spf --version    # superfile 1.5.0
```

## Basic usage

```bash
# launch in current directory
spf

# launch in a specific directory
spf ~/Projects

# launch with a file pre-selected
spf ~/Projects/notes/today.md
```

In the TUI (default keys):

```text
h / l           navigate up / into directory
j / k           move selection down / up
n               new file panel
w               close current file panel
tab / shift-tab cycle file panels
v               select-mode (multi-select)
c / x / p       copy / cut / paste
d               delete (with confirm modal)
ctrl+r          rename
/               filter / fuzzy search in panel
?               help (full hotkey list)
e               open in $EDITOR / configured open_with
```

## When to choose

- **You want a desktop-feeling file manager
  in the terminal without the modal-mode
  learning curve of vifm / nnn / ranger** —
  `spf` ships sensible defaults, an image
  preview that just works, and the keymap is
  visible from `?` rather than memorised.
- **You move between two directories a lot**
  (downloads → project tree, photos → backup)
  — the multi-panel layout is the killer
  feature; one panel per side, copy / move
  between them with one key.
- **You want previews for binary formats out
  of the box** — image, video (first frame),
  PDF (first page) all render inline through
  the terminal's image protocol; no extra
  config to wire up `chafa` / `pistol` /
  `bat`.

## When NOT to choose

- **You live inside a modal-vi-style file
  manager workflow** — use [`nnn`](../nnn/),
  [`ranger`](../ranger/), or [`yazi`](../yazi/);
  `spf` is intentionally not modal.
- **You need a tiny, dependency-free binary
  for ssh-into-a-server use** — [`lf`](../lf/)
  is a single Go binary with a far smaller
  feature surface and lower terminal-protocol
  expectations (no image / video preview
  plumbing assumed).
- **You want a programmable file manager
  scriptable from Lua / shell hooks** —
  [`yazi`](../yazi/) has the most flexible
  plugin system; `spf`'s extension points
  today are the TOML config + `open_with`
  per-extension mapping.

## Why it fits the zoo

Joins the "TUI file manager" cluster
alongside [`yazi`](../yazi/) (Rust, async,
plugin-rich), [`lf`](../lf/) (Go, minimal,
ranger-like), [`nnn`](../nnn/) (C, tiny,
plugin-driven), [`ranger`](../ranger/)
(Python, the original), [`broot`](../broot/)
(tree-summary navigator), [`xplr`](../xplr/)
(Lua-scripted), [`joshuto`](../joshuto/)
(Rust ranger clone), and [`felix`](../felix/)
(minimal Rust). `superfile`'s specific gap
is **opinionated, desktop-app-like
ergonomics with multi-pane and rich previews
out of the box** — the user who wants the
feel of macOS Finder columns or Windows
Explorer panes inside a terminal, with zero
config to get image / video / PDF preview
working.

## Upstream pointers

- Repo: <https://github.com/yorukot/superfile>
- Release notes: <https://github.com/yorukot/superfile/releases>
- License: [MIT](https://github.com/yorukot/superfile/blob/main/LICENSE) (`LICENSE`)
- Docs: <https://superfile.netlify.app/>
- Maintainer: [@yorukot](https://github.com/yorukot)
