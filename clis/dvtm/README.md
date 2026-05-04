# dvtm

> **Tiling window management for the console — dwm's philosophy on
> a TTY** — a ~3000-line C binary (libncurses + libutil only) that
> takes the dwm "no client-side decorations, no config file, edit
> the source" minimalism and applies it to terminal multiplexing:
> N shells (or any commands) tiled into a master + stack layout,
> bottom stack, grid, fullscreen, or fibonacci, switched with one
> keystroke; pair it with `abduco` (same author) and you get the
> session-detach piece tmux bundles, kept as a separate concern.
> Pinned to **v0.15** (released 2020-02-11,
> [LICENSE](https://github.com/martanne/dvtm/blob/v0.15/LICENSE),
> MIT/X Consortium).

Source: <https://github.com/martanne/dvtm>

## TL;DR

`tmux` and GNU `screen` are the obvious answers to "I need
multiple shells in one terminal", but both are **stacked
multiplexers** by default: each new pane gets a slice of one
window, you swap between windows with a prefix key, and the
layout language is its own DSL. `dvtm` flips the model — it is
a **tiling** window manager for the console, in the dwm /
xmonad / i3 lineage. New shells appear in a master area;
additional shells push older ones into a stack column; one
keystroke swaps which layout is active (master+stack, bottom
stack, grid, fibonacci, fullscreen). The binary itself does
*one* thing — multiplex + tile — and explicitly delegates
session detach/reattach to the sister project `abduco`. Result:
a multiplexer you can read end-to-end in an afternoon, that
boots in single-digit milliseconds, and that is configured by
editing `config.def.h` and recompiling, the suckless way.

For an LLM-CLI workflow, the typical shape is `abduco -A work
dvtm` — `abduco` owns the detached session, `dvtm` owns the
in-session tiling — so a remote SSH disconnect leaves an LLM
chat REPL, an editor, a `tail -f agent.log`, and a `git status`
watch all running, exactly where they were, in the same tile
layout.

## Install

```bash
# macOS (Homebrew)
brew install dvtm
# pair with the session manager
brew install abduco

# Debian / Ubuntu
sudo apt install dvtm abduco

# Arch
sudo pacman -S dvtm abduco

# Build from source (any UNIX with libncurses + a C compiler)
git clone https://github.com/martanne/dvtm
cd dvtm && git checkout v0.15
make            # honors PREFIX, CC, CFLAGS
sudo make install

# verify
dvtm -v         # dvtm-0.15 © 2007-2016 Marc André Tanner
```

Hard prereqs: a UTF-8 terminal with 256-color support and
`libncurses` ≥ 5. `dvtm` reads `$TERM`; setting `TERM=dvtm-256color`
in launched shells unlocks per-pane bold/underline/color
(otherwise `dvtm` defaults to a conservative `xterm`-shaped
profile).

## License

MIT / X Consortium License — see
[LICENSE](https://github.com/martanne/dvtm/blob/v0.15/LICENSE).
Permissive; binaries redistribute freely. Because `dvtm` is
configured by editing `config.def.h` and recompiling, downstream
distros ship their own patched builds; the upstream license
covers all of them.

## Hot keybinds

The default modifier is `Ctrl+G` (a deliberate choice: it does
not collide with `bash` / `zsh` readline like `Ctrl+B` / `Ctrl+A`
do). All bindings are `MOD <letter>`.

- `Ctrl+G c` — create a new shell in the current layout
- `Ctrl+G x` — close the focused window (kills the underlying
  process)
- `Ctrl+G j` / `Ctrl+G k` — focus next / previous window
- `Ctrl+G Space` — cycle through layouts (master+stack → bottom
  stack → grid → fibonacci → fullscreen → back)
- `Ctrl+G h` / `Ctrl+G l` — shrink / grow the master area
- `Ctrl+G m` — set the focused window as the master
- `Ctrl+G t` — switch to "tile" layout (master+stack, the
  default)
- `Ctrl+G g` — switch to grid layout
- `Ctrl+G b` — switch to bottom-stack layout
- `Ctrl+G [` — enter copy mode (vi-style scrollback navigation
  + selection); `y` to copy, `q` to exit
- `Ctrl+G ]` — paste the most recent selection into the focused
  window
- `Ctrl+G 1` … `Ctrl+G 9` — jump to window N (the muscle-memory
  binding once you have ≥3 panes)
- `Ctrl+G q` — quit dvtm (closes the session — pair with `abduco`
  for detach instead)
- `Ctrl+G a` — pass a literal `Ctrl+G` to the inner program
  (escape hatch when an editor wants the modifier)

## Why use it

Three reasons to reach for `dvtm` over `tmux`/`screen`:

1. **Tiling-WM ergonomics on the console.** If you already
   think in dwm / xmonad / i3 / Aerospace terms ("master +
   stack, swap with `Mod+Enter`, grow with `Mod+l`"), `dvtm`
   gives you exactly that mental model in a TTY — no plugin,
   no `tmux.conf` translation. Adding a fourth shell is one
   keystroke and the layout adjusts itself; you never resize
   panes by hand.
2. **Suckless-shape codebase.** ~3000 lines of C, one
   `config.def.h` for keybinds + colors + layouts, one binary,
   no Lua / no Python / no plugin runtime. You can read the
   source over lunch, patch it the next morning, and ship a
   custom build that does the one extra thing your workflow
   needs. The opposite of "I will configure tmux for a year
   and still hit a bug in someone else's plugin".
3. **Composable concern split.** `dvtm` does multiplexing +
   tiling. `abduco` does session detach/reattach. `dtach` (a
   third-party peer) is another detach option. You combine
   them — `abduco -A name dvtm` for "tmux-shaped" usage, plain
   `dvtm` inside a `screen` session, plain `dvtm` directly on
   a tty — instead of inheriting one tool's choices. Each
   piece is small enough to audit.

## Vs Already Cataloged

- **Vs [`zellij`](../zellij/):** both are multiplexer
  alternatives but on opposite ends of the surface-area
  spectrum. `zellij` ships visible status bars, KDL layout
  files, a wasm plugin runtime, mouse support, floating panes,
  and built-in session management — pick it when discoverability
  and out-of-the-box features matter. `dvtm` ships ~3000 lines
  of C with `config.def.h` and pairs with `abduco` for
  sessions — pick it when you want the smallest possible
  multiplexer, suckless-shape configurability, and a tiling-WM
  mental model on the TTY.
- **Vs [`tmuxp`](../tmuxp/):** orthogonal. `tmuxp` declares
  `tmux` sessions in YAML/JSON. `dvtm` is the multiplexer
  itself, in a different lineage from `tmux`, and its
  "declarative session" story is `abduco -A name 'dvtm -c
  cmd1 -c cmd2 -c cmd3'` plus a shell function — no separate
  config DSL.
- **Vs [`wtfutil`](../wtfutil/):** orthogonal. `wtfutil` is a
  TUI dashboard of widgets in one terminal. `dvtm` is the
  multiplexer underneath; a common combo is one `dvtm` tile
  running `wtfutil` (dashboard view) and adjacent tiles
  running editor / shell / log tail.

## Caveats

- **No built-in session management.** This is by design —
  `dvtm` exits when its parent shell exits, and detach
  /reattach is delegated to `abduco` or `dtach`. If you forget
  the wrapper and your SSH session drops, every shell inside
  `dvtm` dies. The fix is a one-line wrapper alias:
  `alias work='abduco -A work dvtm'`.
- **No mouse support.** Keyboard only. For workflows that
  expect "click a pane to focus it" or "drag a border to
  resize", this is a hard adjustment — `zellij` and `tmux`
  with `set -g mouse on` both win on this axis.
- **Configuration requires recompilation.** Suckless lineage:
  there is no runtime config file. You edit `config.def.h`
  (then renamed to `config.h` on first build), set keybinds /
  default layouts / status bar format / colors, and `make`. A
  package manager build is the upstream defaults; a custom
  build is your defaults. Distros that patch the build
  (Debian, Arch) effectively ship a third config.
- **Last release was 2020-02-11.** `v0.15` is the most recent
  tag; the project is in maintenance mode (occasional commits
  on `master`, no roadmap). For most use cases this is fine —
  the surface is stable and the bug count is near zero — but
  expect no new features. If you want active development,
  `zellij` is the answer.
- **Status bar is a separate program.** `dvtm` reads from a
  named pipe (`-s /path/to/fifo`) for status text; you wire
  up your own producer (a shell loop with `date`, `uptime`,
  `git status`, etc.) and pipe to that fifo. Out of the box,
  no status line. Power users like this; newcomers expect a
  `tmux`-style preconfigured bar.
- **Copy mode lacks regex search.** Vi-style navigation +
  selection works (`Ctrl+G [` to enter, motion keys, `y` to
  yank), but there is no `/regex` search inside scrollback;
  for that, redirect output to a file or pipe through `less`
  in an adjacent pane.
