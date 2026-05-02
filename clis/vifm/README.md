# vifm

> **Vi-keys file manager** — a ncurses TUI file manager whose
> entire interaction model is `vi` (`hjkl`, `:` command line,
> `/` search, marks, registers, visual mode, `.` to repeat). Two
> panes by default for the venerable Norton/Midnight Commander
> workflow, but every mutation is a vi-style verb. Pinned to
> **v0.14.3** (released 2025-06-04,
> [LICENSE](https://github.com/vifm/vifm/blob/master/COPYING),
> GPL-2.0-or-later).

Source: <https://github.com/vifm/vifm>

## TL;DR

If your muscle memory ends in `:wq`, `vifm` is the file manager
that doesn't make you context-switch. `yy` yanks a file (into a
named register), `dd` cuts, `p` pastes; `:!cmd %f` runs a shell
command on the cursor file; `:filter pattern` hides matches;
`gv` re-selects the last visual block of files. Two synchronized
panes (`<C-w>w` to switch, `=` to mirror, `:sync` to make one
follow the other) give you the dual-pane copy/move workflow that
every Midnight Commander user already knows, but with the keymap
you're already using inside `vim`/`nvim` all day.

Beyond the vi pitch, the practical wins are:

- **`vifmrc` is a real config language**, not a JSON blob.
  `:command`, `:filetype`, `:fileviewer` let you bind viewers
  per glob (`vifm` ships with sensible defaults: `less` for text,
  `mediainfo` for video, image previews via Überzug / sixel /
  Kitty graphics).
- **Built-in archive browsing.** `cd` into a `.tar.gz` / `.zip`
  / `.7z` and navigate it like a directory; FUSE mounts are
  automatic via `:filetype`.
- **`:select` / `:unselect` by glob or grep**, then operate on
  the selection with one verb. No more "shift-click 47 files".
- **Programmable from the outside.** `vifm --remote` sends
  commands to a running instance (`vifm --remote -c ":cd /tmp"`);
  `vifm --choose-files /tmp/out` lets you use vifm as an
  interactive file picker for shell scripts and other TUIs.

## How it compares vs alternatives in this zoo

- vs [`ranger`](../ranger/) — ranger is Python, has miller-columns
  by default (three panes auto-previewing the next level), and a
  much larger Python plugin ecosystem. `vifm` is C, starts in
  ~10ms even on a Pi, defaults to a *symmetric* two-pane layout
  (great for copy/move between two trees), and has a tighter vi
  emulation (registers, marks, `:` history, `:!` substitution
  all behave like vim, not like a vim-flavored Python app).
- vs [`nnn`](../nnn/) — nnn is the minimalist single-pane speed
  king with shell plugins. vifm is denser: more keys to learn, but
  also more vi-style power (visual selection across panes,
  registers, programmable filetype dispatch in `vifmrc` instead of
  shell plugins).
- vs [`yazi`](../yazi/) — yazi is the modern async Rust contender
  with first-class image previews and a plugin manager. vifm is
  the conservative choice: no async, no plugin manager, but it's
  been stable since 2001, builds anywhere there's a C compiler
  and ncurses, and the keymap won't churn between releases.
- vs `mc` (Midnight Commander) — same dual-pane mental model,
  but vi keys instead of F-keys + menus. Pick `mc` if you live
  in F-keys; pick `vifm` if you live in `hjkl`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install vifm

# Linux package managers
# Debian / Ubuntu: apt install vifm
# Fedora: dnf install vifm
# Arch: pacman -S vifm
# Alpine: apk add vifm
# Nix: nix-env -iA nixpkgs.vifm

# Windows (cmd / PowerShell)
# scoop install vifm
# choco install vifm

# from source
git clone https://github.com/vifm/vifm.git && cd vifm
./configure && make && sudo make install

# verify
vifm --version | head -1   # Version: 0.14.3
```

## Examples

```bash
# launch into the current directory (left pane) + $HOME (right pane)
vifm . ~

# inside vifm, day-to-day vi-style verbs:
#   yy          yank file into the unnamed register
#   "ayy        yank into register a
#   dd          cut
#   p           paste into the OTHER pane's cwd (the killer feature)
#   v + motion  visual-select files, then y/d/p as a batch
#   :!less %f   run a shell command on the cursor file (%f = filename)
#   :!less %a   ... on every selected file
#   /pattern    search; n / N to step
#   :filter docx$   hide all .docx files in the current view
#   :sync       make the other pane mirror this pane's cwd
#   gh / gl     parent dir / enter dir (ranger-compat keys)
#   za          toggle hidden files

# use vifm as a fzf-style picker for a shell script:
chosen=$(mktemp)
vifm --choose-files "$chosen" ~/Documents
xargs -a "$chosen" -d'\n' tar czf bundle.tgz
```

## When NOT to reach for vifm

- You don't use vi/vim — the keymap is the entire selling point.
  Use [`yazi`](../yazi/), [`nnn`](../nnn/), or `mc` instead.
- You want async previews of huge directories or remote SSH
  trees — [`yazi`](../yazi/) is purpose-built for that.
- You want a plugin manager and a Lua API — that's
  [`yazi`](../yazi/) again. vifm's extension story is shell
  commands wired in via `vifmrc`, which is intentional but
  limiting if you wanted, say, a fuzzy-finder plugin ecosystem.
