# pizauth

> **A long-running OAuth2 token broker for the command line — keep CLI tools authenticated to IMAP / SMTP / Gmail / Office 365 mail servers without re-doing the device-code dance every hour, and without baking refresh tokens into every config file.**
> A small Rust daemon by Laurence Tratt that runs in the background, holds the
> refresh tokens in memory (or on disk via the OS keyring), and exposes one
> `pizauth show <account>` command that prints the current access token to
> stdout — exactly the shape `mbsync`, `isync`, `msmtp`, `mutt`, `neomutt`,
> `aerc`, and `notmuch` already understand via their `PassCmd` / `tunnel_cmd`
> hooks. Pinned to **pizauth-1.0.11**
> ([COPYRIGHT](https://github.com/ltratt/pizauth/blob/master/COPYRIGHT),
> dual [LICENSE-APACHE](https://github.com/ltratt/pizauth/blob/master/LICENSE-APACHE)
> / [LICENSE-MIT](https://github.com/ltratt/pizauth/blob/master/LICENSE-MIT)).

Source: <https://github.com/ltratt/pizauth>

## TL;DR

Modern mail providers (Gmail, Office 365, Fastmail's OAuth lane, Yahoo) have
deprecated app-passwords for IMAP/SMTP and require OAuth2 with a refresh-token
flow. The plain-text-mail toolchain — `isync`/`mbsync`, `msmtp`, `mutt`,
`neomutt`, `aerc`, `notmuch` — has no native OAuth2 support and instead asks
you for an access token on every connection. Without a broker, you end up
either (a) hand-running a refresh script in a cron job and writing tokens to
disk, or (b) embedding a Python `oauth2.py` snippet in every config and hoping
it doesn't race when `mbsync -a` opens 12 parallel connections.

`pizauth` is the daemon-shaped answer: one config file
(`~/.config/pizauth.conf`) declares each account's `auth_uri` /
`token_uri` / `client_id` / `scopes`, you run `pizauth refresh <account>`
once to do the browser-flow handshake, and from that point on every other
tool just calls `pizauth show <account>` and gets a current access token in
under a millisecond. The daemon refreshes tokens in the background before
they expire, retries on transient network failure, and falls back to
prompting via `notify-send` (or whatever `notify_cmd` you configure) when a
refresh token finally expires and a human needs to re-consent.

## Install

```bash
# Homebrew (macOS / Linux)
brew install pizauth

# Cargo (any platform with a Rust toolchain)
cargo install --locked pizauth

# Arch Linux (AUR)
yay -S pizauth

# verify
pizauth --version
# pizauth 1.0.11
```

After install, drop a `~/.config/pizauth.conf` (see upstream `pizauth.conf.example`),
start the daemon as a user service (`systemctl --user start pizauth`, or a
launchd plist on macOS), and run `pizauth refresh <account>` once per
account to seed the refresh token.

## Usage

```bash
# 1) One-time consent for an Office 365 mailbox
pizauth refresh work-o365
# pizauth opens the auth URL in your browser; you sign in;
# pizauth captures the redirect on its loopback listener and stores
# the refresh token. Done — repeat only when the refresh token expires.

# 2) Wire mbsync to use it (~/.mbsyncrc)
#   PassCmd "pizauth show work-o365"
#   AuthMechs XOAUTH2

# 3) Ad-hoc check from any script
ACCESS_TOKEN=$(pizauth show work-o365)
curl -H "Authorization: Bearer $ACCESS_TOKEN" \
     https://outlook.office365.com/api/v2.0/me/messages?$top=1
```

## Niche & tradeoffs

`pizauth` lives in the narrow but high-value slot of "OAuth2 token broker
for headless CLI tools," distinct from:

- **Browser-side password managers** ([`rbw`](../rbw/) for Bitwarden,
  `pass` for GPG, `bw` for hosted Bitwarden) — those store *static
  secrets*; pizauth specifically manages *short-lived access tokens
  refreshed from a long-lived refresh token*. The two compose cleanly:
  store your client secret in `pass` / `rbw`, let pizauth handle the
  refresh-token dance.
- **Cloud credential brokers** ([`granted`](../granted/),
  [`aws-vault`](../aws-vault/)) — those broker *cloud IAM* (STS / SSO /
  IAM Identity Center) for AWS / Azure / GCP. pizauth brokers
  *application OAuth2* (Gmail, O365, generic OAuth2 IdPs). Same
  refresh-and-cache shape, completely different scope / audience /
  threat model.
- **Per-tool plugins** (`oauth2ms`, `mutt_oauth2.py`, `gmail-oauth2`)
  — single-purpose Python scripts that work but have to be
  re-implemented per protocol; pizauth replaces the whole zoo with one
  daemon and one `show` interface that any tool with a `PassCmd`
  can hit.

The right mental model is "**`ssh-agent`, but for OAuth2 access tokens**":
a long-running per-user process that holds the sensitive material in
memory, talks to clients over a per-user socket, and answers one
question — "give me a current credential for *this* account" — in
constant time. Pair with [`himalaya`](../himalaya/) for the modern
TUI/CLI mail client side, with [`aerc`](../aerc/) when a TUI mail
client with native account isolation is the goal, with [`mbsync`](https://isync.sourceforge.io/)
+ [`notmuch`](../notmuch/) (when present) for the classic local-IMAP
mirror + tag-search workflow.

Caveats — (1) refresh-token rotation policies vary by IdP, and some
providers (notably Office 365 Personal) silently invalidate refresh
tokens after a few weeks of inactivity; configure `notify_cmd` so
pizauth can prompt for re-consent rather than silently failing in the
mail client. (2) The default config keeps refresh tokens in process
memory only — if pizauth crashes or the box reboots, you re-run
`pizauth refresh`; opt into the persistent `token_event_cmd`-driven
keyring storage if reboot-survivable tokens matter. (3) The loopback
redirect URI requires the IdP to allow `http://127.0.0.1:<port>/`
as a registered redirect — Google and O365 do, some niche IdPs do
not, in which case you fall back to the `auth_uri`'s out-of-band
copy-paste mode. (4) Single-user, per-machine — there is no shared
mode for serving tokens to multiple operators from one daemon; that
is by design (the threat model is "my laptop, my mail").
