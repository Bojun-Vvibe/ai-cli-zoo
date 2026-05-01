# sshx

- **Repository:** https://github.com/ekzhang/sshx
- **Latest version:** v0.4.1
- **License:** MIT — verified at [`LICENSE`](https://github.com/ekzhang/sshx/blob/main/LICENSE) (raw: https://raw.githubusercontent.com/ekzhang/sshx/main/LICENSE)
- **Niche:** Instant, end-to-end-encrypted, **multi-user collaborative
  terminal sharing over a public web URL** — no SSH server on the
  host, no port forwarding, no account on either side.

## What it does

`sshx` is a single static Rust binary plus a hosted (or
self-hostable) relay. Run `sshx` in any terminal and it prints a
`https://sshx.io/s/<id>#<key>` URL — opening that URL in a browser
gives anyone with the link a real `xterm.js` view of the same shell,
with multiple cursors, named participants, and a shared scrollback.

```
# Share the current shell — prints a URL with the decryption key in the fragment
sshx

# Read-only mode (great for screencasts / on-call walk-throughs)
sshx --enable-readers

# Run a specific command instead of inheriting $SHELL
sshx -- htop

# Self-host the relay (single binary, no DB)
sshx-server --listen 0.0.0.0:8051

# Point the client at your own relay
sshx --server https://sshx.example.com
```

## Why interesting

The ergonomics of "let me see your terminal real quick" have been
broken forever. The options were:

- **`tmux` + SSH** — needs an SSH account on the host, needs the
  other person to have a terminal, needs you to share a key, and
  doesn't survive NAT.
- **`tmate`** — the closest prior art; great, but session sharing is
  terminal-only (no browser), the encryption story predates modern
  E2EE expectations, and it doesn't do multi-cursor collaborative
  editing of the same shell.
- **Screen-share over Zoom** — works, costs you a meeting, can't
  copy-paste a stack trace out of the other person's terminal,
  bandwidth-heavy.

`sshx` is the version where the URL goes in chat, the recipient
clicks it, and they're typing in your shell five seconds later — no
account, no install, no port forward, no QR code. The encryption key
lives in the URL fragment (`#...`), so the relay (even
`sshx.io`) sees only ciphertext: it cannot read the session, and a
shoulder-surfer who screenshots the *prefix* of the URL still cannot
join. The web client is `xterm.js` with real multi-cursor — two
people can edit the same `vim` buffer with named cursors, and a
mentor can highlight the line they're talking about without taking
control.

The single-static-binary self-host story (`sshx-server`, no
database, no Redis, no nothing) makes it usable inside a private
network where `sshx.io` is blocked or against policy: stand the
relay up on an internal VM, point the client at it with `--server`,
done.

## Pairs well with

- [`tmate`](../tmate/) — the terminal-native cousin. Reach for
  `tmate` when your audience is also at a terminal and you want
  classic `tmux` semantics; reach for `sshx` when you need a browser
  URL, multi-cursor, or a read-only screencast link.
- [`ttyd`](../ttyd/) — exposes a terminal as a long-lived web URL on
  a server you control; lacks the ad-hoc "share this shell *right
  now* for ten minutes" UX and the multi-cursor collaboration model.
- [`asciinema`](../asciinema/) + [`agg`](../agg/) — record-and-replay
  cousin. `sshx` is *live* collaboration; `asciinema` is asynchronous
  playback. Different tools for different incident postures.

## When to skip

- You need persistent, audited, account-bound remote access to a
  server — that's what real SSH (with [`teleport`](https://goteleport.com)
  or your bastion of choice) is for; `sshx` is an ephemeral
  collaboration channel, not an access-control system.
- The host is fully air-gapped with no outbound HTTPS — `sshx` needs
  to reach the relay (yours or `sshx.io`); use `tmux` over a VPN
  instead.
- You want a recorded artifact you can paste into a postmortem —
  `asciinema` records to a local `.cast` file; `sshx` sessions are
  ephemeral by design (the relay holds nothing after disconnect).
