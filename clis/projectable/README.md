# projectable

> Snapshot date: 2026-05. Upstream: <https://github.com/dzfrias/projectable>

**A `lazygit`-shaped TUI for the project itself, not the git
history.**
projectable opens in the current directory and gives you a
file-tree pane, a preview pane, a marks pane, and a command-palette
pane that all share keybindings and a single config file. You
navigate the tree with `hjkl`, preview files with `bat`-style
syntax highlighting, run user-defined shell commands against the
selected path (`build`, `test`, `lint`, `open in editor`), bookmark
files you keep coming back to, and grep-jump across the tree —
without leaving the terminal or opening a full editor.

## Repo + version + license

- Repo: <https://github.com/dzfrias/projectable>
- Latest release: **`1.3.2`** (2025-01-13)
- License: **MIT** —
  <https://github.com/dzfrias/projectable/blob/main/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `main`
- Language: Rust

## Install

```bash
# Cargo
cargo install projectable

# Homebrew tap
brew install dzfrias/tap/projectable

# Or grab a prebuilt binary from GitHub Releases
# (Linux x86_64, macOS x86_64/aarch64).

# Open in the current directory
prj

# Open against an explicit path
prj ~/code/my-service

# All the keybindings and per-project commands live in one TOML:
#   <project>/.projectable.toml   (per-repo)
#   ~/.config/projectable/config.toml   (global)
```

## Niche

The "**file-tree-first project TUI with per-project commands and
marks**" slot. Where [`yazi`](../yazi/) and
[`superfile`](../superfile/) are *general-purpose file managers*
optimised for browsing a whole filesystem, projectable is scoped
to *one project at a time*: it expects a `.git` (or a
`.projectable.toml`) at the root, treats that as the universe,
and gives you commands defined *for that project*. Useful for:

- **Polyglot monorepos** — different per-directory commands
  (`cargo test` in `crates/*`, `pnpm test` in `web/*`,
  `pytest` in `services/*/python`) bound to the same keys, picked
  by the path of the currently-selected file.
- **Reading unfamiliar codebases** — file-tree + syntax-highlighted
  preview + fuzzy-find + marks pane is the shape you want for
  "spend an afternoon understanding this repo" without committing
  to opening it in an editor.
- **Bouncing between a few hot files** — the marks pane (`m` to
  toggle) is the projectable equivalent of editor bookmarks, and
  it persists across sessions per-project.
- **Shelling out without losing context** — `!` opens an arbitrary
  shell command with the selected path interpolated; the TUI keeps
  the tree state when you return.

## Why it matters

- **Per-project `.projectable.toml`** — checked into the repo,
  shared with the team. Define `[[commands]]` blocks with `name`,
  `key`, `cmd` (with `{path}` interpolation), `confirm`, and
  `working_dir`, and every contributor gets the same one-keystroke
  build / test / lint / format actions inside the TUI without
  needing to remember the project's specific incantations.
- **`bat`-style preview pane** — syntax-highlighted file preview
  with theme support, image preview via Sixel where the terminal
  supports it, and graceful fallback for binary files (size + magic
  bytes summary instead of garbage).
- **Marks that persist** — marked files survive across sessions
  per-project, stored next to the config; tab-cycle through marks
  with `Tab` / `Shift-Tab` to bounce between the four files you're
  actively editing.
- **Fuzzy file jump (`/`) and grep jump (`?`)** — `/` is a
  fzf-style filename filter over the visible tree, `?` shells out
  to ripgrep and jumps to the matching file. Both keep the tree
  oriented so you don't lose context.
- **Single Rust binary, no daemon** — projectable is one process
  per terminal pane; no background indexer, no IPC. The startup
  cost is the file-tree walk, which is bounded by `.gitignore` so
  it stays fast on large repos.
- **Honest scope** — projectable is a *project navigator*, not an
  editor and not an IDE. It does not edit files in-place (it shells
  out to `$EDITOR`), it does not run a language server, it does not
  manage git history (use [`lazygit`](../lazygit/) /
  [`gitui`](../gitui/) / [`gitu`](../gitu/) for that). The win is
  composition: projectable + your editor + lazygit in three panes
  is a complete project shell.
- **Maintenance caveat** — `1.3.2` (2025-01) is the current
  release at snapshot time; project is small and the maintainer
  ships incremental fixes rather than a steady cadence. The config
  format and on-disk marks file are stable, so switching away later
  is a `cargo uninstall projectable` and a `rm
  .projectable.toml`.
