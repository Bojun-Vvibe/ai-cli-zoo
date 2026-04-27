# atuin

> **Magical shell history** — replaces the flat `~/.bash_history` /
> `~/.zsh_history` line-buffer with a SQLite-backed, fuzzy-searchable,
> context-aware (cwd, exit code, duration, hostname, session) history
> store, optionally end-to-end encrypted and synced across machines
> through a self-hostable server, with a full-screen TUI bound to
> `Ctrl+R` that finally makes shell history a queryable database
> instead of a scrollback. Pinned to **v18.15.2** (commit
> `de4a1555b9023859c5afd00075068b4df1d8ce98`,
> [LICENSE](https://github.com/atuinsh/atuin/blob/main/LICENSE), MIT).

Source: <https://github.com/atuinsh/atuin>

## TL;DR

`atuin` is the shell-history upgrade you didn't realise you needed.
After `atuin init zsh|bash|fish|nushell|xonsh` adds a one-line hook
to your rc file, every command you run is captured into a local
SQLite store at `~/.local/share/atuin/history.db` along with the
working directory, exit code, wall-clock duration, hostname,
session id, and (optionally) git context. `Ctrl+R` opens a
full-screen TUI with fuzzy filtering, scope toggles
(`session` / `directory` / `host` / `global`), live preview, and
filter modes (`prefix` / `fuzzy` / `regex` / `fulltext`) that beat
the stock `bck-i-search` by a wide margin. `atuin search foo`,
`atuin stats`, and `atuin history list --cwd` make the same store
queryable from scripts. The optional `atuin sync` layer
end-to-end-encrypts the history with a passphrase-derived key
(libsodium / `XChaCha20-Poly1305`) and pushes it to a free
hosted instance or a `cargo install atuin-server` you run
yourself, so all your machines see the same history without ever
trusting the server with the plaintext.

## Install

```bash
# one-shot installer (detects shell, writes the init line, downloads the binary)
curl --proto '=https' --tlsv1.2 -LsSf https://setup.atuin.sh | sh

# Homebrew
brew install atuin

# Cargo
cargo install atuin

# manual init in your rc file (zsh shown)
echo 'eval "$(atuin init zsh)"' >> ~/.zshrc

# import existing history once
atuin import auto

# verify
atuin --version       # atuin 18.15.2
```

Runs entirely local by default — no account, no network call. The
sync layer is opt-in (`atuin register` / `atuin login` against the
default hosted server, or `ATUIN_SYNC_ADDRESS=https://your-server`
against a self-hosted one).

## One Concrete Example

```bash
# 1. install + init (one time)
brew install atuin
echo 'eval "$(atuin init zsh)"' >> ~/.zshrc
exec zsh
atuin import auto                       # backfill from ~/.zsh_history

# 2. day-to-day: Ctrl+R now opens the TUI
#    typing "psql" filters across history fuzzily
#    Tab cycles scope: session -> directory -> host -> global
#    Enter executes; right-arrow places on the prompt for editing

# 3. ask questions about your own shell history from a script
atuin search --cmd "kubectl" --cwd ~/code/myrepo --limit 20
atuin stats               # most-used commands, top dirs, time spent in shell
atuin history list --cwd ~/code/myrepo --format json | \
  jq '[.[] | select(.exit != 0)] | length'   # count failed commands in this repo

# 4. opt into encrypted sync (optional)
atuin register -u me -e me@example.com    # hosted free tier
# OR self-host:
#   docker run -d -p 8888:8888 ghcr.io/atuinsh/atuin server start
#   ATUIN_SYNC_ADDRESS=https://atuin.internal atuin login -u me
atuin key                              # prints recovery key — save it
atuin sync                             # push local history; pull from other hosts
```

## Niche It Fills

**Shell history as a real local database, not a flat tail-only
log.** Stock shell history loses your exit codes, your working
directories, your timing, and your context the moment a line
scrolls off the buffer; `atuin` keeps all of it in a queryable
SQLite store and gives you a fuzzy TUI in front of it. For an
operator who hops between many directories and machines (and an
LLM-CLI workflow that is *constantly* re-running variations of
the same command), this changes shell history from "scrollback"
into "personal command database."

## Vs Already Cataloged

- **Vs [`zoxide`](../zoxide/):** `zoxide` is the smart-`cd`
  primitive (frecency over directories). `atuin` is the smart-
  `Ctrl+R` primitive (frecency + context over commands). They
  compose — both go in the same `eval` block in your rc file.
- **Vs [`fzf`](../fzf/):** `fzf` ships a generic fuzzy finder
  that *can* be wired to history (`Ctrl+R` widget), but it
  reads the same flat history file with no per-command context.
  `atuin` *replaces* that pipeline with a SQLite-backed store
  that records cwd / exit / duration. Use `fzf` for generic
  picker UIs; use `atuin` for the history-specific UI.
- **Vs [`mcfly`](../mcfly/):** the closest peer — both are
  smart-`Ctrl+R` replacements with SQLite + neural ranking.
  `atuin` adds end-to-end encrypted multi-host sync (the
  killer differentiator) and a self-hostable server; `mcfly`
  is local-only and lighter to install. Pick `atuin` if you
  use more than one machine.
- **Vs [`lnav`](../lnav/):** `lnav` is a log viewer for log
  *files*; `atuin` is a history store for *your shell*.
  Different layers, no overlap.

## Caveats

- The `Ctrl+R` rebinding is the point — the first time it feels
  unfamiliar versus stock `bck-i-search`. `atuin init zsh
  --disable-up-arrow` is the recommended posture if the
  Up-arrow rebinding (history walk) is too aggressive for muscle
  memory.
- Sync is **opt-in** but the hosted free tier is a Pangea-managed
  service (free up to a generous quota); the *encryption* is
  client-side libsodium so the operator never sees plaintext, but
  the operator can see ciphertext blob counts / sync timestamps.
  Self-host with `cargo install atuin-server` + Postgres if even
  that metadata is a concern (single binary, 1 dependency).
- The recovery key (`atuin key`) is the *only* way to decrypt your
  history on a new machine — losing it after enabling sync means
  starting over. Save it to your password manager the moment you
  enable sync.
- Capture happens at `preexec` / `precmd` hook boundaries — a
  command that overwrites the same hook (some shell frameworks,
  `direnv`, custom prompt setups) can collide. The README has a
  troubleshooting section if commands stop being captured.
- The local SQLite store grows ~unbounded; `atuin search --before
  '1y ago' --delete` (or a periodic `atuin store rebuild`) keeps
  it tidy if you care.
