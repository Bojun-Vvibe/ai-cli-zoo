# mosh

## What it does
A remote-shell client/server pair (`mosh-client` + `mosh-server`) that replaces
the interactive-SSH terminal with the **State Synchronization Protocol** (SSP)
over UDP datagrams. The wrapper script `mosh user@host` opens a one-shot SSH
connection, has the remote side spawn `mosh-server` on a free UDP port in
60000–61000, exchanges an AES-128-OCB session key, then drops SSH entirely and
keeps a UDP roaming session alive. Because SSP synchronizes *terminal screen
state* instead of replaying a keystroke byte stream, mosh predicts your local
edits and underlines them until the server confirms — so SSH on a 400 ms
satellite link feels like a 400 ms shell, but mosh feels like a local one. The
client survives IP address changes (Wi-Fi → LTE → tether), laptop suspend, and
hours of disconnection without losing the session, because UDP has no
connection state to break.

## Why it's interesting
Different shape from `ssh` / `eternal-terminal` / `tmate` / `zellij --attach`.
SSH is TCP keystroke replay — one dropped packet stalls the head-of-line, one
network change kills the socket. `eternal-terminal` rebuilds session resumption
on top of TCP and a relay process; `tmux` / `zellij` give you detach/reattach
but only if you remembered to start them and only on the same SSH socket.
`mosh` solves the *transport* layer: it is the only one of these that handles
"I closed the lid, took a train, opened the lid in another country" without
any wrapper, multiplexer, or relay server. Choose it when you SSH from a
laptop that roams networks, when latency is high enough that local echo
matters, or when you want resumable terminals without running `tmux`. Do
**not** choose it when you need port forwarding, X11 forwarding, SFTP, or
agent forwarding — mosh deliberately delegates all of that back to SSH and
only handles the interactive shell.

## Niche category
Roaming-tolerant remote shell — UDP state-sync replacement for the interactive SSH terminal.

## Repo
https://github.com/mobile-shell/mosh

## Version pinned
`mosh 1.4.0` (latest tagged release, `v1.4.0`)

## License
- SPDX: `GPL-3.0-or-later`
- License file in upstream repo: `COPYING`

## Install
```sh
brew install mosh
# Debian/Ubuntu: apt install mosh
# Fedora:        dnf install mosh
# Server side needs the same package; the wrapper auto-spawns mosh-server over SSH.
# Open UDP 60000-61000 on the server's firewall.
```

## Usage examples
```sh
# Drop-in replacement for ssh
mosh user@host.example.com

# Use a non-default SSH port for the bootstrap handshake
mosh --ssh="ssh -p 2222" user@host.example.com

# Force a specific UDP port range (firewall-friendly)
mosh --server="mosh-server new -p 60005" user@host.example.com

# Combine with tmux so the terminal *and* the shell history both survive
mosh user@host -- tmux new -A -s main

# Show predictive local echo always (default is "adaptive" — only on slow links)
mosh --predict=always user@host

# What's actually running on the server side after you connect?
ssh user@host 'pgrep -af mosh-server'
```
