# zellij

> **Terminal multiplexer with batteries included** — a single Rust
> binary that gives you `tmux`-class pane / tab / session management
> on top of any shell, but ships with a *visible* status / tab bar by
> default, a `Ctrl+G` "lock-mode" that shows every keybind on screen,
> floating panes, a per-session layout language (KDL), and a
> WebAssembly plugin runtime so third-party panes (file browsers,
> status widgets, completion popups) drop in without recompiling the
> host. Pinned to **v0.44.1** (commit
> `f2af4fe5e35a553aa9b0494ca86d85b73c625c66`,
> [LICENSE](https://github.com/zellij-org/zellij/blob/main/LICENSE.md),
> MIT).

Source: <https://github.com/zellij-org/zellij>

## TL;DR

`zellij` is what you reach for when you want `tmux`'s
"my session survives the SSH drop" durability without `tmux`'s
"I have to memorize 40 invisible keybinds and write a 200-line
`.tmux.conf` before it stops fighting me" onboarding cliff. Run
`zellij` with no args to attach the default session (or create
one), `zellij --layout default` to boot a named layout, `zellij
attach <name>` to reattach a detached session. Out of the box you
get: a status bar that names the current mode (`NORMAL` / `PANE` /
`TAB` / `RESIZE` / `MOVE` / `SCROLL` / `SEARCH` / `SESSION` /
`LOCKED`), a tab bar across the top, mouse support, and a help
strip at the bottom listing the keys for the current mode — so the
"how do I split a pane again" recall problem solves itself. Panes
split with `Ctrl+P` then `n` (new) / `d` (down) / `r` (right),
floating panes toggle with `Ctrl+P` then `w`, the scrollback gets
its own mode (`Ctrl+S`) with regex search, and `Ctrl+O` then `d`
detaches the session so a remote disconnect leaves your work
running. Layouts are KDL files (`~/.config/zellij/layouts/*.kdl`)
that declare panes / tabs / running commands declaratively — boot
a "log + editor + REPL + git status" workspace with one command.

## Install

```bash
# Homebrew (macOS / Linux)
brew install zellij

# Cargo
cargo install --locked zellij

# Linux package managers
# Arch: pacman -S zellij
# Nix: nix-env -iA nixpkgs.zellij
# Fedora: dnf copr enable varlad/zellij && dnf install zellij

# from a release tarball (any OS)
ZELLIJ_VERSION=$(curl -s "https://api.github.com/repos/zellij-org/zellij/releases/latest" | grep -Po '"tag_name": "v\K[^"]*')
curl -Lo zellij.tar.gz "https://github.com/zellij-org/zellij/releases/latest/download/zellij-aarch64-apple-darwin.tar.gz"
tar xf zellij.tar.gz zellij
sudo install zellij /usr/local/bin/

# verify
zellij --version    # zellij 0.44.1
```

Zero config required for the first run — first invocation creates
`~/.config/zellij/` with sensible defaults. Optional:
`zellij setup --dump-config > ~/.config/zellij/config.kdl` to
extract the full default config as a KDL file you can edit.

## License

MIT — see [LICENSE](https://github.com/zellij-org/zellij/blob/main/LICENSE.md).
Permissive, no attribution required for binaries; redistribute
freely.

## Hot keybinds

The defaults split keybinds into "modes" — press the mode key
first, then a single letter. The `LOCKED` mode (`Ctrl+G`)
disables every zellij binding so the underlying shell sees raw
input (useful for `vim` / `helix` users whose own keymaps clash).

- `Ctrl+G` — toggle locked mode (passthrough every key to the
  inner program; press `Ctrl+G` again to release)
- `Ctrl+P` — pane mode: `n` new pane, `d` split down, `r` split
  right, `x` close, `f` toggle fullscreen, `w` toggle floating, `e`
  toggle embedded / floating, `z` toggle frames, arrow keys focus
- `Ctrl+T` — tab mode: `n` new tab, `x` close tab, `r` rename, `s`
  toggle sync (broadcast input to all panes), arrow keys / `1-9`
  switch tab
- `Ctrl+N` — resize mode: arrow keys grow / shrink the focused
  pane in that direction
- `Ctrl+H` — move mode: relocate the focused pane within the tab
- `Ctrl+S` — scroll mode: `j` / `k` line scroll, `Ctrl+F` /
  `Ctrl+B` page, `s` enter search, `e` edit scrollback in `$EDITOR`
- `Ctrl+O` — session mode: `d` detach session, `w` switch session,
  `c` toggle session manager
- `Ctrl+Q` — quit zellij (closes the session)
- `Alt+n` — new pane in the same direction as last split (the
  "muscle-memory" binding once you have a workflow)
- `Alt+[` / `Alt+]` — previous / next tab
- `Alt+f` — toggle floating pane (without first entering pane
  mode — the keybind that gets used most)

## Why use it

Three things `zellij` does that `tmux` and `screen` do not, that
pay back the switching cost permanently:

1. **Visible mode + keybind hints by default.** The status bar
   names the current mode and the bottom strip lists the keys
   available *right now* — so a workflow like "split a pane,
   resize it, detach the session" is discoverable instead of
   memorized. `Ctrl+G` to lock mode means even users whose inner
   editor has clashing keymaps can pass everything through with
   one keystroke. Onboarding a new teammate to a shared workflow
   stops requiring a cheat-sheet.
2. **Layouts as KDL files.** A layout is a declarative
   `.kdl` file that names tabs, splits panes, and runs initial
   commands. `zellij --layout dev` boots an "editor pane + log
   tail + test-runner pane + git status pane" workspace as one
   command, reproducible across machines, diffable in git. The
   `tmux` equivalent (a `tmuxinator` YAML file or a hand-written
   shell script of `send-keys` calls) is more fragile and not
   first-class.
3. **WebAssembly plugin runtime.** Plugins are wasm modules that
   run in a sandboxed pane with explicit permissions (file system,
   network, command exec). `zellij`'s built-in tab bar, status
   bar, and session manager are themselves plugins — third-party
   plugins (`zjstatus` for a richer status bar, `zellij-forgot`
   for a keybind helper, `monocle` for fullscreen-with-search) are
   `cp foo.wasm ~/.config/zellij/plugins/` and a layout reference.
   The `tmux` plugin model (shell scripts + `tmux` config commands)
   is more limited.

For an LLM-CLI workflow that wants "an editor in pane 1, an LLM
chat REPL in pane 2, a `tail -f` of the agent log in pane 3, and
a `git status` watch in pane 4" as a one-command workspace,
zellij's KDL layouts collapse the setup from "five `tmux send-keys`
commands" to one `zellij --layout llm-dev`.

## Vs Already Cataloged

- **Vs `tmux` (not cataloged):** the closest peer. `tmux` has
  ~15 years of maturity, more plugins, and is preinstalled on
  most Linux distros and macOS via Homebrew. Pick `tmux` when
  you need maximum portability (every server has it) or when
  your config investment is sunk. Pick `zellij` when the
  visible-modes + KDL-layouts + wasm-plugins combination is
  worth the switching cost.
- **Vs [`atuin`](../atuin/):** orthogonal — `atuin` is a shell
  history database, `zellij` is a terminal multiplexer. They
  compose: run `atuin` inside any zellij pane, the history is
  shared across panes via the shared SQLite store.
- **Vs [`lazygit`](../lazygit/):** orthogonal — `lazygit` is a git
  TUI that runs *inside* a pane. A common zellij layout: editor
  pane on the left, `lazygit` floating-pane on a hotkey for git
  operations.
- **Vs [`posting`](../posting/):** orthogonal — `posting` is a TUI
  HTTP client. Run it in a zellij pane next to your editor; the
  KDL layout can boot both at once.

## Caveats

- **Sessions are per-installation.** `zellij list-sessions` only
  sees sessions on the current host — there is no equivalent of
  `tmux -L <socket>` for multi-socket isolation, and no built-in
  session sync across machines. Pair with `mosh` if you need
  network-resilience on top.
- **KDL is its own config language.** Newcomers expect TOML / YAML
  / JSON; KDL has its own grammar (nodes with children, `=`-less
  attributes, `r"raw strings"`). The `zellij setup --dump-config`
  output is the cheat-sheet — read it before hand-rolling complex
  layouts.
- **Plugin ecosystem is smaller than tmux's.** The wasm runtime is
  newer; the plugin catalog is dozens not hundreds. For a
  workflow that depends on a specific `tmux` plugin (e.g.
  `tmux-resurrect` for reboot-survival), check whether a zellij
  equivalent exists before switching — the answer is sometimes
  "not yet".
- **Default keybind set conflicts with `bash` / `zsh`.** `Ctrl+P`
  / `Ctrl+N` / `Ctrl+S` are bound by zellij and by readline
  (previous-command / next-command / forward-search-history). The
  `LOCKED` mode (`Ctrl+G`) is the escape hatch but easy to forget
  in the first week. Remap or live with `Ctrl+G` muscle memory.
- **Floating panes do not persist across detach / reattach** in
  the same way tiled panes do — the layout is restored, but a
  floating pane that was open at detach time may come back as a
  tiled pane on reattach. Keep load-bearing panes tiled.
