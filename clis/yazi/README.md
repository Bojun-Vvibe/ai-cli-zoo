# yazi

> **Async terminal file manager** — a single Rust binary that opens
> a `ranger`-style three-column TUI (parent dir / current dir /
> preview) on top of any directory tree, where every filesystem
> operation (copy, move, rename, bulk-rename, archive, extract,
> tag, sort, search) is one or two keystrokes, the preview pane
> renders images inline (sixel / kitty / iterm2 protocols), and a
> Lua plugin runtime + on-disk DSL for keybinds and filetype
> openers makes the whole thing extensible without recompiling.
> Pinned to **v26.1.22** (commit
> `ea6f30b09b3147889ee8e2cda9e89c6a8c3a8e26`,
> [LICENSE](https://github.com/sxyazi/yazi/blob/main/LICENSE),
> MIT).

Source: <https://github.com/sxyazi/yazi>

## TL;DR

`yazi` is what you reach for when `cd` + `ls` + `mv` + `cp` is
finally costing more keystrokes than it saves on a tree you do
not have memorized. `yazi` opens a three-column TUI: the parent
directory on the left, the focused directory in the middle (with
a cursor you move with `j` / `k` and arrow keys), and a preview
of the focused entry on the right (file contents for text, image
for image formats, archive listing for `.tar` / `.zip`, dir listing
for directories). Navigation is `vim`-flavored (`h` / `j` / `k`
/ `l`, `gg` / `G` for top / bottom, `/` for in-dir search,
`Ctrl+f` / `Ctrl+b` for page up / down). The killer feature versus
`ranger` / `nnn` / `lf` is **everything is async** — directory
reads, file previews, image rendering, archive scanning, plugin
calls all run on a Tokio runtime, so `yazi` never freezes when you
cursor onto a 10 GB directory or a slow network mount; the preview
pane shows a loading state and updates when ready instead of
blocking the UI. Image previews use the terminal's native graphics
protocol (`kitty`, `iterm2`, `sixel`, `chafa` fallback) and render
in-cell — no external `ueberzug` daemon. Bulk-rename (`a` to add
to the selection, then `:` for command palette → `rename`) opens
the selected names in `$EDITOR`, you edit, save, and yazi applies
the diff. Archives extract with `x` and create with `:archive`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install yazi

# Cargo
cargo install --locked yazi-fm yazi-cli

# Linux package managers
# Arch: pacman -S yazi
# Nix: nix-env -iA nixpkgs.yazi
# Fedora: dnf copr enable lihaohong/yazi && dnf install yazi

# from a release tarball (any OS)
curl -Lo yazi.zip "https://github.com/sxyazi/yazi/releases/latest/download/yazi-aarch64-apple-darwin.zip"
unzip yazi.zip
sudo install yazi-*/yazi /usr/local/bin/
sudo install yazi-*/ya /usr/local/bin/

# verify
yazi --version    # Yazi 26.1.22
```

Optional shell wrapper for "`cd` to where yazi exited":

```bash
function y() {
  local tmp="$(mktemp -t "yazi-cwd.XXXXXX")"
  yazi "$@" --cwd-file="$tmp"
  if cwd="$(command cat -- "$tmp")" && [ -n "$cwd" ] && [ "$cwd" != "$PWD" ]; then
    builtin cd -- "$cwd"
  fi
  rm -f -- "$tmp"
}
```

After this, `y` opens yazi and on quit (`q`) the parent shell `cd`s
to wherever you ended up — the workflow tmux/zellij users want.

## License

MIT — see [LICENSE](https://github.com/sxyazi/yazi/blob/main/LICENSE).
Permissive, no attribution required for binaries; redistribute
freely.

## Hot keybinds

These pay back the learning cost in the first hour:

- `h` / `j` / `k` / `l` — left up down right (left = parent dir,
  right = enter focused dir / open file)
- `g g` / `G` — top of dir / bottom of dir
- `/` / `?` — search forward / backward in current dir; `n` / `N`
  next / previous match
- `space` — toggle selection on focused entry; `v` enter visual
  selection mode; `V` invert selection
- `y` — yank (copy); `x` — cut; `p` — paste; `P` — paste forced
  overwrite; `d` — move to trash; `D` — delete permanently with
  confirm
- `r` — rename in place; `a` — create new file (`a/` for new
  directory); `:` — command palette (rename, archive, chmod, …)
- `Tab` — toggle preview pane focus; `M` — toggle hidden files;
  `s` — sort menu (name / size / mtime / ctime / extension);
  `f` — filter (live regex filter on current dir)
- `o` / `O` — open with default opener / pick opener; `Enter` —
  open with first matching opener rule
- `c c` — copy path to clipboard; `c d` — copy parent dir to
  clipboard; `c f` — copy filename only
- `t` — toggle tab; `Ctrl+t` — new tab; `1-9` — switch to tab N
- `q` — quit; `Q` — quit without writing cwd-file (so the `y`
  shell wrapper does not `cd`)

## Why use it

Three things `yazi` does that `ranger` / `nnn` / `lf` do not, that
pay back the switching cost:

1. **Async runtime means no blocking.** Cursoring onto a directory
   with 100k entries, a 10 GB tarball, or a network mount that's
   slow to `stat` does not freeze the UI — yazi shows a loading
   state and updates when the read completes. `ranger` (Python)
   and `lf` (Go, but synchronous IO) both stutter on the same
   inputs.
2. **First-class image preview without `ueberzug`.** `yazi` talks
   the terminal's native graphics protocol (`kitty`, `iterm2`,
   `sixel`) directly — no separate daemon, no X11 dependency, no
   "preview disappears when I scroll" bug. On terminals without
   graphics support, `chafa` fallback shows a unicode-block
   approximation. Photo / screenshot triage in a directory is one
   keystroke (`l` to focus an image, preview pane renders it).
3. **Lua plugin runtime + on-disk keybind / opener DSL.** Keybinds
   live in `~/.config/yazi/keymap.toml`, openers in `yazi.toml`,
   plugins drop into `~/.config/yazi/plugins/<name>.yazi/`. A plugin
   is a Lua module that registers commands; the community catalog
   covers archive-passthrough, git-integration (status icons in
   the listing), `chmod` modal, smart-enter (cd into dirs, open
   files), and theme packs. `ranger`'s Python plugin model is more
   powerful but requires a Python runtime in your terminal; `yazi`
   plugins are sandboxed Lua and ship in the binary.

For an LLM-CLI workflow that produces dozens of generated files
in a session and needs quick triage (preview, rename, move into a
project tree, delete the wrong ones), `yazi`'s preview-on-cursor
+ bulk-select + bulk-rename collapses the "alt-tab between
terminal and Finder / Files" loop.

## Vs Already Cataloged

- **Vs [`zellij`](../zellij/):** orthogonal — `zellij` is a terminal
  multiplexer (panes / tabs / sessions), `yazi` is a file manager
  that runs *inside* a pane. A common workflow: zellij layout
  with one pane running `yazi` for navigation, another pane for
  the editor or REPL.
- **Vs [`atuin`](../atuin/):** orthogonal — `atuin` is shell
  history, `yazi` is a file manager. They compose: search yazi's
  command palette through atuin's `Ctrl+R` if you launched yazi
  from a shell that has atuin bound.
- **Vs [`harlequin`](../harlequin/):** orthogonal — `harlequin` is
  a SQL IDE for `.db` files, `yazi` is a file manager. Use yazi to
  find the `.db` file, press the right opener key, harlequin
  launches with that file as argument.
- **Vs [`posting`](../posting/):** orthogonal — `posting` is a TUI
  HTTP client, `yazi` is a file manager. They share the "nice TUI
  in a terminal pane" niche but solve different problems.

## Caveats

- **Image previews require a graphics-capable terminal** — `kitty`,
  `wezterm`, `iterm2`, modern `foot` / `ghostty`, or any terminal
  that supports sixel. `gnome-terminal` / `Terminal.app` (macOS
  default) do not — yazi falls back to `chafa` unicode blocks
  there, which is functional but ugly. Switch terminal if image
  triage is a load-bearing use case.
- **The `ya` CLI is a separate binary** for IPC into a running yazi
  instance (`ya emit reveal /path/to/file` to focus a path from
  another pane). The Homebrew formula installs both, but the
  cargo install needs `yazi-fm` *and* `yazi-cli` — easy to forget
  one and wonder why `ya` is not on PATH.
- **Bulk-rename via `$EDITOR` is line-order-sensitive** — yazi
  pairs the Nth original line with the Nth edited line. Reorder
  or delete lines and the rename mapping shifts. Read the diff
  preview before confirming.
- **Trash on Linux requires `trash-cli`** for `d` to send to
  `~/.local/share/Trash/`; without it, `d` errors out and you
  must use `D` (permanent delete with confirm). On macOS the
  built-in trash is used directly.
- **Plugin API is still pre-1.0-stable** — the project's calendar-
  versioning scheme (`v26.1.22` = year 2026, month 1, release 22)
  signals fast iteration; plugins published against an older
  release sometimes need a one-line patch when yazi bumps. Pin
  plugins with explicit commits in `~/.config/yazi/package.toml`.
- **No first-class git integration in core** — listing icons by
  git status, staging from yazi, etc., live in plugins
  (`git.yazi`, `starship.yazi`). Install one if needed; the core
  binary is intentionally narrow.
