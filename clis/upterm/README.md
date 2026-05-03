# upterm

> **Pair-program over the public internet by sharing your local
> shell through an SSH reverse-tunnel** — `upterm host` spawns a
> shell, registers a one-time session ID with a relay
> (`uptermd.upterm.dev` by default, or self-hosted), and prints a
> `ssh <id>:<token>@uptermd.upterm.dev` URL the remote pair pastes
> into their terminal to attach. Authorized SSH public keys gate
> the join, the host can flip read-only vs read-write per peer,
> and a tmux pane underneath multiplexes screen + scrollback so
> the guest sees the same view the host sees. Pinned to **v0.22.0**
> ([LICENSE](https://github.com/owenthereal/upterm/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/owenthereal/upterm>

## TL;DR

`upterm host -- bash` opens a shareable session; relay returns
the connect string; collaborator runs `ssh ...` and lands in the
same shell. No browser, no plugin, no inbound port on the host
(the host dials *out* to the relay over SSH on 22 / 2222 and
listens for forwarded connections), works from coffee-shop NAT
and corporate firewalls without touching router config.

## Install

```bash
# Homebrew (macOS / Linux)
brew install owenthereal/upterm/upterm

# Go
go install github.com/owenthereal/upterm/cmd/upterm@v0.22.0

# Pre-built release tarball
curl -LO https://github.com/owenthereal/upterm/releases/download/v0.22.0/upterm_darwin_arm64.tar.gz
tar xf upterm_darwin_arm64.tar.gz
sudo install upterm /usr/local/bin/

# verify
upterm version    # 0.22.0
```

`upterm` requires an SSH agent / key on the host. By default it
loads `~/.ssh/id_*` and uses GitHub-published public keys
(`--github-user <login>`) or GitLab / SourceHut equivalents to
authorize joiners.

## License

Apache-2.0 — see
[LICENSE](https://github.com/owenthereal/upterm/blob/master/LICENSE).
Permissive, patent-grant included; safe to embed in commercial
internal tooling.

## One Concrete Example

```bash
# 1. share a bash shell with a teammate identified by their GitHub keys
upterm host --github-user alice -- bash
# prints:  ssh TOKEN:abc123@uptermd.upterm.dev

# 2. read-only screen-share for a demo (audience cannot type)
upterm host --read-only -- bash

# 3. share a long-running command, not an interactive shell
upterm host -- tail -f /var/log/app.log

# 4. allow multiple authorized peers (intersection of all flags)
upterm host --github-user alice --github-user bob -- zsh

# 5. self-hosted relay (data-residency / air-gapped corp network)
upterm host --server ssh://upterm.internal.corp:2222 -- bash

# 6. inspect live sessions (host side)
upterm session list
upterm session info <id>
```

## Niche It Fills

**Zero-infrastructure live terminal pairing across NAT / firewall
boundaries with SSH-grade auth.** The same problem `tmate`
solves, but using stock OpenSSH on the relay (no patched tmux
fork on the wire), reachable by any `ssh` client (no special
client binary on the joiner side), and gated by the same public
keys you already publish on GitHub / GitLab.

## Why use it

1. **Joiner needs only `ssh`.** No browser tab, no Electron app,
   no "install our client first." Works from a phone with
   Termius, from a server-room jump host, from a Windows box
   with the OpenSSH that ships in Windows 10+.
2. **Auth piggybacks on your existing public keys.** `--github-user
   alice` looks up `https://github.com/alice.keys` and authorises
   exactly those keys — no new credential store, no shared
   secret to leak.
3. **Self-hostable in one binary.** `uptermd` is the relay; one
   process behind a public IP plus a host SSH key, and the
   "share my shell" feature now lives entirely on infra you
   control (compliance / air-gap / data-residency).

## Vs Already Cataloged

- **Vs [`tmate`](../tmate/):** tmate is the original "pair on a
  tmux session over a relay" tool and ships its own patched tmux;
  the joiner uses a shortened ssh URL but the wire protocol is
  tmate's. `upterm` uses stock SSH end-to-end on the wire (the
  relay is an SSH server doing reverse tunnels) and stock tmux
  for the multiplexer underneath, so the joiner can be *any*
  SSH client and the relay is independently auditable.
- **Vs `ssh -R` to a bastion + `screen -x`:** the manual
  reverse-tunnel + shared-screen recipe works but requires a
  bastion you SSH into, a shared Unix account, manual session
  cleanup, and no read-only mode. `upterm` collapses all of that
  into one command and adds per-peer read-only + GitHub-key
  authorisation.
- **Vs [`teleport`](../teleport/):** teleport is the full
  enterprise zero-trust access platform (SSO, audit log, session
  recording, RBAC); upterm is the "I want to pair on this
  terminal for the next 20 minutes" point tool. Different
  weight class.

## Caveats

- **Default relay is a third party.** `uptermd.upterm.dev` is run
  by the maintainer; for any session that involves
  customer-data-shaped output or production credentials, pass
  `--server ssh://relay.you.run:2222`. The README explicitly
  recommends self-hosting for sensitive use.
- **Keys, not passwords.** No `--password` flag — joiners
  without an SSH key on the host's allow-list cannot connect.
  This is correct behaviour but trips up first-time users who
  expected a shared link to "just work."
- **tmux dependency.** The host wraps the shared command in tmux
  for resize handling and scrollback; on minimal containers /
  Alpine images you may need `apk add tmux` first.
- **No built-in session recording.** If you need the audit trail
  (who joined when, what was typed), wrap the host shell in
  `script -f session.log` or use a heavier tool (teleport, the
  enterprise option).
