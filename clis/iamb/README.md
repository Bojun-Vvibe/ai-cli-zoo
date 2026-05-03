# iamb

> **A modal, Vim-keybind-native Matrix client TUI in
> Rust** — connect to any Matrix homeserver
> (matrix.org, your self-hosted Synapse / Dendrite /
> Conduit, EMS), and the entire UX is `:`-commands
> (`:join`, `:leave`, `:invite`, `:dms`, `:rooms`,
> `:replied`, `:forget`) plus normal-mode motion over
> the room list, message history, and reply tree —
> with end-to-end encryption (E2EE), threads,
> attachments, image previews via the Kitty / Sixel /
> iTerm2 graphics protocols, and desktop notifications
> all working out of the box. Pinned to **v0.0.11**
> ([LICENSE](https://github.com/ulyssa/iamb/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/ulyssa/iamb>

## TL;DR

If you live in tmux + neovim and resent every flatpak
Element launch, `iamb` is the Matrix client you wanted.
It is built on `matrix-rust-sdk` (the same crypto stack
Element-X uses), so encrypted DMs, verified devices,
cross-signed identities, and threaded conversations all
work the same as the GUI clients. The difference is the
UX: there is no mouse-driven sidebar; you `:join
#rust:matrix.org`, you `[count]j` down the message
buffer, you `r` to reply, `o` to open the focused link,
`/` to search, and `:dms` to switch to the DM list. The
config (`~/.config/iamb/config.toml`) is one TOML file
per profile (you can run multiple Matrix accounts in
the same binary by passing `--profile work` /
`--profile personal`).

## Install

```bash
# Homebrew (macOS / Linux)
brew install iamb

# Cargo (any platform with a Rust toolchain)
cargo install --locked --version 0.0.11 iamb

# Pre-built binary (Linux x86_64 musl tarball)
curl -L https://github.com/ulyssa/iamb/releases/download/v0.0.11/iamb-x86_64-unknown-linux-musl.tgz \
    | tar xz
sudo install iamb /usr/local/bin/

# Linux x86_64 .deb
curl -LO https://github.com/ulyssa/iamb/releases/download/v0.0.11/iamb-x86_64-unknown-linux-musl.deb
sudo dpkg -i iamb-x86_64-unknown-linux-musl.deb

# Linux aarch64 .rpm
curl -LO https://github.com/ulyssa/iamb/releases/download/v0.0.11/iamb-aarch64-unknown-linux-gnu.rpm
sudo rpm -i iamb-aarch64-unknown-linux-gnu.rpm

# Arch Linux (community)
pacman -S iamb

# Nix (flake)
nix run github:ulyssa/iamb

# verify
iamb --version    # iamb 0.0.11
```

First run drops you into a profile-creation prompt;
after that, `~/.config/iamb/config.toml` looks roughly
like:

```toml
default_profile = "personal"

[profiles.personal]
user_id = "@you:matrix.org"

[profiles.work]
user_id = "@you:your-homeserver.example"
url = "https://matrix.your-homeserver.example"
```

Login uses interactive password or
[SSO](https://github.com/ulyssa/iamb/blob/main/docs/login.md)
the first time, then a stored session token in
`~/.local/share/iamb/`.

## Use it for

```bash
# Launch with the default profile
iamb

# Pick a profile explicitly
iamb --profile work

# Inside the TUI:
# :join #rust:matrix.org          – join a public room
# :rooms                          – list joined rooms
# :dms                            – list 1:1 conversations
# :spaces                         – list Matrix spaces
# :invite @alice:matrix.org       – invite to current room
# :replied                        – jump to the message the selected message replied to
# :upload ~/screenshot.png        – attach a file
# :verify                         – verify a device (E2EE)
# :forget                         – forget all left rooms (clean up sidebar)
# :room topic show                – inspect / change room metadata
# :keys                           – show key bindings
# :q                              – quit

# Normal-mode motion (same as Vim):
# j/k    – next / prev message in scrollback
# G/gg   – bottom / top of timeline
# /      – search messages
# o      – open URL in $BROWSER (or focused image with image viewer)
# r      – reply to focused message
# d      – redact (delete) focused message (if you sent it)
# m      – open message thread

# Window management (Vim-style splits):
# :vsplit #anotherroom:matrix.org   – open a second room in a vertical split
# Ctrl-w h/j/k/l                    – move between windows
# :tabnew                           – new tab (each tab can hold multiple windows)
```

Image previews are off by default; turn them on per
profile in `config.toml`:

```toml
[profiles.personal.settings]
image_preview = { protocol = { type = "kitty" } }   # or "sixel", "iterm2", "halfblocks"
notifications = { enabled = true, show_message = true }
```

`iamb` will then render Matrix image attachments inline
in the timeline — the only catalog TUI Matrix client
that does this without spawning an external viewer.

## Why include it in a CLI catalog

1. **It is the cleanest "Matrix in a terminal" the
   ecosystem has produced.** Older TUI Matrix clients
   (`gomuks`, `weechat-matrix`, `nheko`'s curses port)
   work, but none commit to *modal* editing the way
   `iamb` does. If you already think in `:wq` / `daw` /
   `cw`, the muscle memory transfers wholesale —
   message buffers behave like read-only Vim buffers,
   the prompt line behaves like the Vim command line.
   The author maintains the underlying `modalkit` crate
   precisely so this stays consistent.
2. **End-to-end encryption is first-class.** Built on
   `matrix-rust-sdk` (the SDK Element-X is built on),
   so `:verify` cross-signs devices the same way
   Element does, and encrypted rooms decrypt without
   special configuration. This is the single biggest
   reason previous-generation TUI Matrix clients fell
   off — they predated the modern crypto SDK and never
   caught up. `iamb` is on Matrix SDK 0.14 as of
   v0.0.11.
3. **Stays out of the way.** No XDG-portal popups, no
   Electron startup cost, no "your message hasn't sent
   yet" spinner blocking input. The whole binary is a
   ~5-25MB static Rust executable; SSH into a server
   and run `iamb` there to read your DMs without
   forwarding X11 / Wayland / a browser. Image
   previews require a graphics-protocol-capable
   terminal but are an opt-in feature, not a required
   one.

For an LLM-CLI workflow, `iamb` is the path to wire
chatops back into your terminal: a self-hosted Matrix
homeserver + a bot that posts agent transcripts gives
you a searchable, encrypted, multi-device-synced log of
every `aider` / `codex` / `claude-code` session, viewable
inside the same tmux you're coding in.

## Vs Already Cataloged

- **Vs [`hut`](../hut/):** orthogonal — `hut` is the
  CLI for `sourcehut` (todo, paste, build, mail).
  `iamb` is realtime chat. They live next to each
  other in a "everything I do at the terminal" tmux
  layout: `iamb` in one pane, `hut todo list` in
  another.
- **Vs [`halloy`](../halloy/):** closest peer in
  spirit — both are Rust TUI/GUI chat clients with
  modern crypto / protocol support. `halloy` speaks
  IRC; `iamb` speaks Matrix. Pick by which network
  your community lives on. Many Rust / NixOS / Linux
  channels have bridges, so you can often run either.
- **Vs [`tut`](../tut/):** orthogonal — `tut` is a
  Mastodon TUI; `iamb` is a Matrix TUI. Same author
  family of "real social networks in the terminal", but
  Mastodon (broadcast-style) and Matrix (room/DM-style)
  are different products.
- **Vs [`himalaya`](../himalaya/) /
  [`aerc`](../aerc/) / [`mbsync`](../mbsync/):**
  orthogonal — those are email; `iamb` is realtime
  chat. They co-exist in the same TUI workflow.

## Caveats

- **0.x version line.** v0.0.11 is the most recent as
  of January 2026. The project has been releasing
  steadily for 2+ years and is the recommended TUI
  Matrix client by `matrix.org`'s own client list, but
  the version number reflects "I don't promise config
  stability across minor versions" — pin a specific
  version when packaging.
- **First sync is slow on large accounts.** If you
  have 200+ joined rooms across multiple
  homeservers, the initial `iamb` launch can take
  30–90 seconds while it pulls room state and
  decrypts history. Subsequent launches are fast
  (incremental sync only).
- **Modal-only is a learning cliff.** No mouse, no
  click-to-focus (mouse scroll is opt-in via config).
  If you do not already have Vim muscle memory,
  expect a real onboarding curve. `:keys` shows the
  bindings; `man iamb` documents every command.
- **Image previews require a graphics-capable
  terminal.** Kitty / WezTerm / Ghostty / iTerm2 /
  any Sixel-aware terminal works; xterm + tmux 3.4+
  with passthrough works for Kitty graphics; older
  terminals fall back to placeholder text.
- **Apache-2.0 license.** Permissive; redistributable
  inside proprietary products with attribution.
- **Project is single-maintainer.** Active and shipping
  releases (~quarterly), but bus-factor is real for
  any community-supported chat client. The reliance on
  `matrix-rust-sdk` (Element-funded) for the heavy
  crypto / sync work mitigates this — most of the
  surface area is shared with Element-X.
