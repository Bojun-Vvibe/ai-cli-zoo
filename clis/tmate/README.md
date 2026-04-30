# tmate

- **Repo:** https://github.com/tmate-io/tmate
- **Latest version:** 2.4.0 (release `2.4.0`)
- **HEAD on `master`:** `3e12f55`
- **License:** ISC (with BSD-style components from upstream tmux) — [COPYING](https://github.com/tmate-io/tmate/blob/master/COPYING)
- **Category:** terminal sharing / remote pairing

A **fork of `tmux` that adds instant terminal sharing over SSH**.
Run `tmate` and the tool spins up a tmux-compatible session, opens
an outbound TLS connection to a relay (the public `tmate.io`
service by default, or a self-hosted `tmate-ssh-server` on your
own infrastructure), and prints two pairs of credentials: a
**read-only** SSH command and a **read-write** SSH command, each
in both raw-SSH and `https://` web-client form. Anyone you share
the credentials with — a teammate behind corporate NAT, a contractor
on the other side of the planet, an on-call engineer on a phone
hotspot — can attach in one paste, with no port-forwarding, no
VPN, no inbound firewall changes on either end. Because the
underlying session *is* tmux, every existing tmux keybinding,
config file, status line, and pane layout works unchanged; the
remote viewer sees pixel-identical output and, in read-write mode,
shares a real keyboard with the host. Sessions survive client
disconnects, can be re-attached from a new device, and end when
the host process exits.

## Install

```bash
# macOS
brew install tmate

# Debian / Ubuntu
sudo apt-get install -y tmate

# Arch
sudo pacman -S tmate

# Linux — prebuilt static binary from a release
curl -L -o tmate.tar.xz https://github.com/tmate-io/tmate/releases/download/2.4.0/tmate-2.4.0-static-linux-amd64.tar.xz
tar xJf tmate.tar.xz && sudo mv tmate-2.4.0-static-linux-amd64/tmate /usr/local/bin/

# verify
tmate -V
```

## Usage examples

```bash
# Start an interactive shared session — prints SSH + web URLs to stderr,
# then drops you into a normal tmux-style session you can pair on:
tmate

# Headless / scripted mode — useful for CI debugging or one-shot
# "give me a shell on the runner" workflows. Wait for the session to
# come up, then print only the read-write SSH string for piping to
# Slack, a chat bot, or a job summary:
tmate -F new-session -d
tmate wait tmate-ready
tmate display -p '#{tmate_ssh}'
```

## Why it matters

Pair-programming, incident response, and "look at my screen for
ten seconds" debugging are bottlenecked by *access*, not by
intent. Screen-sharing tools (Zoom, Meet, Tuple) require both
sides to have the app, a good network, and a willingness to grant
remote control to a GUI; SSH-based sharing requires inbound
network reachability the host usually does not have. `tmate`
collapses both problems into a single outbound TLS connection
and a string you can paste into any chat: the viewer needs only
`ssh` (preinstalled on every macOS / Linux box and most Windows
since the OpenSSH client became standard) or a browser, and the
host needs only to be able to reach the public internet. Combined
with the read-only credential, it is also the cleanest way to
broadcast a **live, copy-pasteable terminal** during a workshop,
a war-room incident, or an interview — viewers see exactly what
you typed, can scroll back through tmux history, and never accidentally
take control. For CI runners, dropping `tmate` into a failing job
gives a real shell on the actual runner in under five seconds,
which is dramatically better than the "add `echo` statements,
push, wait twelve minutes, repeat" loop that is otherwise the
only option on hosted CI.
