# toot

> **Mastodon from the terminal** — a Python CLI + full-screen TUI
> that talks to any Mastodon-compatible instance (Mastodon, GoToSocial,
> Pleroma, Akkoma, Pixelfed read-only) for reading the timeline,
> posting, replying, boosting, following, searching, and managing
> multiple accounts, with an interactive `toot tui` mode that turns
> the same shell into a usable Mastodon client. Pinned to **0.52.1**
> ([LICENSE](https://github.com/ihabunek/toot/blob/master/LICENSE),
> GPL-3.0).

Source: <https://github.com/ihabunek/toot>

## TL;DR

`toot` is the keyboard-driven, scriptable counterpart to the web
Mastodon UI. `toot login` walks an OAuth flow against any instance
and stores the token in `~/.config/toot/config.json`; from then on
`toot timeline` streams your home feed as text, `toot post "hello
fediverse"` toots, `toot tui` opens a full-screen ncurses-style
client with vim-ish key bindings (`j`/`k`, `r`eply, `b`oost,
`f`avourite, `c`ompose), and every read verb (`toot search`, `toot
notifications`, `toot thread`, `toot whois`) is also a one-shot
JSON-emitting command suitable for piping into `jq` or wrapping
in a script. Multi-account is a first-class verb (`toot
activate-account`), so you can flip between a personal and a
work persona without re-authing.

## Install

```bash
# Homebrew (macOS / Linux)
brew install toot

# pipx (recommended for Python isolation)
pipx install toot

# pip (per-user)
pip install --user toot

# Arch
pacman -S toot

# Debian / Ubuntu
apt install toot

# Nix
nix-env -iA nixpkgs.toot

# verify
toot --version    # toot 0.52.1
```

```bash
# first-time setup against any Mastodon-compatible instance
toot login              # walks OAuth in your browser, stores token
toot whoami             # confirms account
```

## License

GPL-3.0 — see
[LICENSE](https://github.com/ihabunek/toot/blob/master/LICENSE).
Copyleft: redistributions of modified `toot` itself must remain
GPL-3.0; using `toot` as a CLI from your own scripts does not
infect those scripts.

## One Concrete Example

```bash
# 1. read the home timeline (last 20)
toot timeline -c 20

# 2. post — text, a CW, and a media attachment
toot post --visibility unlisted \
  --spoiler-text "long: ops post-mortem" \
  --media ~/screenshots/dashboard.png \
  "We rolled back the deploy at 14:02 UTC. Postmortem at the link."

# 3. reply to a specific status by ID (IDs come from `toot timeline`)
toot post --reply-to 110482918273645511 "+1, this matches what I saw"

# 4. search and pipe to jq (toot's read verbs all support --json)
toot search "openapi 3.1" --json | \
  jq '.statuses[] | {acct: .account.acct, content: .content}'

# 5. follow / unfollow by handle
toot follow @user@mastodon.example
toot unfollow @user@mastodon.example

# 6. multi-account: log in to a second instance and switch
toot login                         # second OAuth flow
toot accounts                      # list configured accounts
toot activate-account me@second.social
toot whoami                        # now reflects the second account

# 7. full-screen TUI mode
toot tui
# j/k navigate, r reply, b boost, f favourite, c compose, q quit

# 8. stream the live timeline (long-running)
toot timeline --once     # one fetch, exits
# or pair with watch: watch -n 30 'toot notifications -c 10'
```

## Niche It Fills

**A scriptable client for an open-protocol social network.** The
fediverse is HTTP + ActivityPub all the way down, but most
clients are GUI-only and most "Mastodon CLI" projects are
abandoned wrappers around 2017-era APIs. `toot` is the one that
keeps up with the current Mastodon API surface, supports the
non-Mastodon implementations (GoToSocial, Pleroma) that share
the protocol, and treats both human use (TUI) and automation
use (JSON-emitting subcommands) as first-class. For an LLM-CLI
workflow it's the difference between "I can post status updates
from a cron job and read replies from a script" and "I have to
maintain my own ActivityPub HTTP client".

## Why use it

1. **Same binary, two modes.** `toot post` for scripts and
   shortcuts; `toot tui` for actually reading the feed without
   leaving the terminal. No second tool to install, no protocol
   re-implementation, one config file.
2. **Multi-instance, multi-account, OAuth-correct.** `toot
   login` walks the OAuth code flow per instance and stores the
   tokens correctly scoped; switching accounts is one verb, not
   an env-var dance.
3. **JSON output on every read verb.** `--json` (or the default
   for the API-shaped verbs) means `toot search`, `toot
   timeline`, `toot notifications`, `toot thread`, etc. compose
   with `jq` without parsing the human-readable output. This is
   the lever that turns "fediverse client" into "fediverse
   automation primitive" — incident bots, release announcers,
   hashtag watchers, etc.

## Vs Already Cataloged

- **Vs [`tut`](../tut/):** the closest peer. `tut` is a
  Go-based **TUI-first** Mastodon client (richer key bindings,
  slicker rendering, but no scriptable one-shot CLI verbs);
  `toot` is **CLI-first** (every verb works headless and emits
  JSON) and includes a TUI as a secondary mode. Pick `tut` if
  you only ever read interactively; pick `toot` if you also
  want to wire Mastodon into scripts / cron / CI.
- **Vs [`gh`](../gh/) / [`glab`](../glab/) / [`forgejo`](../forgejo/):**
  same shape (CLI for a federated protocol with a JSON API),
  different network. Use `toot` when the source-of-truth is a
  Mastodon-compatible server, not a forge.
- **Vs [`himalaya`](../himalaya/) / [`mbsync`](../mbsync/):**
  email is a different federated protocol with a different
  identity model and different verbs (folders, threads, MIME);
  no overlap beyond "terminal client for a federated network".
- **Vs `curl` against the Mastodon API directly:** `curl` works
  fine for one-shot reads but you'd have to implement OAuth,
  pagination, multi-account token storage, and content rendering
  yourself. `toot` is what you'd build if you tried.

## Caveats

- **Python runtime dep.** `toot` is a Python package; on a fresh
  box without Python you're installing CPython too. The `pipx`
  install path keeps it isolated; the Homebrew bottle bundles a
  Python interpreter. The Go-based `tut` is the leaner choice
  if disk + interpreter footprint matters.
- **TUI is functional, not flashy.** `toot tui` covers the daily
  verbs (read / reply / boost / favourite / compose) but doesn't
  render images inline (it shows the URL), doesn't have the
  column-based layout of e.g. Pinafore / Elk, and doesn't stream
  in real time inside the TUI (you refresh manually). For
  passive reading it's adequate; for power-user fediverse-as-
  workflow it can feel sparse.
- **No Bluesky / AT Proto support.** Despite the name overlap in
  the social-network-CLI niche, `toot` is Mastodon (ActivityPub)
  only. Bluesky requires a different client.
- **Media-upload size limits are per-instance.** `--media` sends
  the file as multipart upload and inherits the instance's
  configured max size (default 8 MB on stock Mastodon, often
  16 MB or more on community instances). Large videos may need
  pre-compression with `ffmpeg`.
- **Token storage is plaintext on disk.** `~/.config/toot/config.json`
  holds OAuth bearer tokens unencrypted. Treat it like any other
  CLI credential file (mode 600, not in dotfile-sync repos
  without an exclude). On macOS, layering `pass` / Keychain
  isn't built in.
