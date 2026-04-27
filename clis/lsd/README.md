# lsd

> **The next-gen `ls(1)` — colourful, icon-aware, tree-capable**, a
> Rust rewrite that picks up where `exa` was abandoned and runs in
> parallel to its successor [`eza`](../eza/). Pinned to **v1.1.5**
> (2024-09-22 release; HEAD on `master` is the active line),
> [LICENSE](https://github.com/lsd-rs/lsd/blob/master/LICENSE),
> Apache-2.0.

Source: <https://github.com/lsd-rs/lsd>

## TL;DR

`ls` was designed for a 1978 VT100 and shows it: alignment is
columnar, colours are an opt-in `--color=auto` flag, sizes are
either bytes or `-h`-rounded with a single character of suffix,
and there is no notion of "this is a Rust project, that is a
Node project" — every entry is a name and a mode. `lsd` keeps
the same flag surface (`-l`, `-a`, `-h`, `-S`, `-t`, `-r`) and
adds: file-type icons from a Nerd-Font glyph map, per-file-type
colour theming via a YAML config, human dates that auto-pick
relative ("2 days ago") vs absolute, a `--tree` mode that mirrors
`tree(1)` with depth limits and gitignore awareness, git-status
column (`-G`) that shows `?`/`M`/`A`/`D` next to each entry, and
`--total-size` to roll directory sizes recursively. Cross-platform
(Linux, macOS, FreeBSD, Windows). Renders correctly in `tmux`,
`zellij`, `kitty`, `wezterm`, `iTerm2`, and Windows Terminal —
falls back gracefully when `TERM` does not advertise truecolor or
when no Nerd Font is installed.

## Install

```bash
# Homebrew (macOS / Linux)
brew install lsd

# Cargo
cargo install --locked lsd

# Linux package managers
# Arch:   pacman -S lsd
# Debian/Ubuntu: apt install lsd        # 22.04+
# Fedora: dnf install lsd
# Nix:    nix-env -iA nixpkgs.lsd

# Windows
# scoop install lsd
# winget install lsd-rs.lsd

# release tarball (any OS / arch)
curl -Lo lsd.tar.gz "https://github.com/lsd-rs/lsd/releases/download/v1.1.5/lsd-v1.1.5-aarch64-apple-darwin.tar.gz"
tar xf lsd.tar.gz --strip-components=1 && sudo install lsd /usr/local/bin/

# verify
lsd --version    # lsd 1.1.5
```

A Nerd Font is **strongly** recommended (FiraCode Nerd Font,
JetBrainsMono Nerd Font, Hack Nerd Font); without one the icon
column renders as `?` boxes. `--icon never` disables icons
entirely for SSH-into-busybox or CI log scenarios.

## License

Apache-2.0 — see
[LICENSE](https://github.com/lsd-rs/lsd/blob/master/LICENSE).
Permissive with patent grant; suitable for vendoring into
corporate base images.

## One Concrete Example

```bash
# 1. Drop-in: alias ls=lsd in your shell rc and the muscle memory keeps working
alias ls='lsd'
ls -lah ~/Projects                    # icons + colours + human sizes

# 2. Tree view with depth limit (replaces `tree -L 2`)
lsd --tree --depth 2 src/

# 3. Tree that respects .gitignore (replaces `tree -I 'node_modules|target'`)
lsd --tree --depth 3 --ignore-glob 'node_modules|target|.git'

# 4. Git status column — see modified/added/deleted next to each file
lsd -lG

# 5. Roll directory sizes recursively (default `ls -lh` shows 4096 for dirs)
lsd -lh --total-size ~/Downloads

# 6. Sort by size, descending, human-readable (largest dirs first)
lsd -lhSr --total-size

# 7. Long format with relative dates
lsd -l --date relative ~/Projects

# 8. JSON-like grouped output for piping into jq via --classic + parsing
lsd -l --classic         # disables icons/colours for stable scripting

# 9. Theme it: drop ~/.config/lsd/colors.yaml and ~/.config/lsd/icons.yaml
mkdir -p ~/.config/lsd && lsd --config-file ~/.config/lsd/config.yaml -l
```

## Niche It Fills

**The "modern `ls`" slot, Apache-2.0 flavour.** The space has
three live options: `exa` (abandoned, MIT, last release 2021),
[`eza`](../eza/) (active fork of `exa`, MIT, the de-facto
successor), and `lsd` (independent rewrite, Apache-2.0, focused
on icons/themes over `exa`-style git-aware tabular output). `lsd`
wins when you want (a) Apache-2.0 licensing rather than MIT, (b)
a stronger default icon set, (c) a YAML theming surface that
recolours by file extension without recompiling. `eza` wins when
you want git-blame-in-`ls`, `--git-repos` summary mode, or a
slightly faster `--tree` over very large trees. Use both — they
do not conflict — and pick per-host based on which package
manager has the more recent build.

## Why use it

1. **Icons that mean something.** Each file extension maps to
   a Nerd Font glyph (`.rs` → ferris, `.ts` → TypeScript logo,
   `.dockerfile` → whale, `Cargo.toml` → cargo box). At a glance
   a directory listing tells you "this is a Rust project with a
   Dockerfile and a README" without reading a single filename.
2. **Tree mode that respects `.gitignore`.** `tree(1)`'s `-I`
   flag takes a `|`-separated glob list and you re-type
   `node_modules|target|.git|dist|.next|.venv` every time. `lsd
   --tree --ignore-glob` picks up `.gitignore` patterns
   automatically when run inside a git repo.
3. **YAML theming without recompile.**
   `~/.config/lsd/colors.yaml` and `~/.config/lsd/icons.yaml`
   override per-extension colour and glyph at runtime; ship the
   same files across machines via dotfiles and every host looks
   the same.

For an LLM-CLI workflow `lsd --tree --depth N` is the canonical
"give the model a one-screen view of this project" command — pipe
it into the prompt context window and the model sees structure
without you hand-curating a file list.

## Vs Already Cataloged

- **Vs [`eza`](../eza/):** the closest neighbour. `eza` is the
  community-maintained `exa` successor (MIT, ~2k stars/month
  active); `lsd` is the independent Rust rewrite (Apache-2.0).
  Feature overlap is ~80%. `eza` has the richer git surface
  (`--git`, `--git-repos`) and a faster wide-tree renderer;
  `lsd` has the better default icon coverage and the YAML
  theming knobs. License preference often decides — pick `lsd`
  in Apache-2.0 estates, `eza` in MIT-friendly ones.
- **Vs [`broot`](../broot/):** orthogonal. `broot` is an
  interactive tree navigator with fuzzy search and a verb DSL
  (`:rm`, `:cp`, `:mv` from inside the tree); `lsd` is a
  one-shot `ls` replacement. Use `broot` when you want to
  *navigate*, `lsd` when you want a *snapshot*.
- **Vs [`yazi`](../yazi/):** also orthogonal. `yazi` is a
  full-screen file manager TUI with image previews, archive
  browsing, and async I/O; `lsd` prints to stdout and exits.
  Compose them: `yazi` for interactive triage, `lsd` for
  scripts and `cd && lsd -lh`.

## Caveats

- **Nerd Font required for icons.** Without a Nerd Font, the
  icon column shows `?` boxes. Add `--icon never` to suppress,
  or install a Nerd Font and configure your terminal.
- **No git blame column.** `eza --git --long` shows per-file
  staging status *and* short-SHA of last touch; `lsd -lG` shows
  status only. Reach for `eza` if you want the blame column.
- **Tree mode is not a `find` replacement.** `lsd --tree` walks
  a single directory tree top-down and renders it; for "find
  every file modified in the last 7 days" use [`fd`](../fd/).
- **`--total-size` is recursive and slow on huge trees.**
  Rolling sizes for a 5M-file `node_modules` tree takes
  noticeable wall-clock time; use [`dust`](../dust/) for the
  "what's eating my disk" question instead.
- **Config file location is platform-specific.** macOS:
  `~/.config/lsd/`; Linux: `$XDG_CONFIG_HOME/lsd/`; Windows:
  `%APPDATA%\lsd\`. Symlink in dotfile setups.
