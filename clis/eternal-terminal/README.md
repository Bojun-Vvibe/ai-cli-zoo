# eternal-terminal

> Snapshot date: 2026-05. Upstream: <https://github.com/MisterTea/EternalTerminal>

**A persistent, roaming SSH replacement (`et`) — survives network
drops, laptop sleep, IP changes, and Wi-Fi handoffs without
killing your shell, and round-trips native scrollback +
`Ctrl-C` to the local terminal so the experience is "ssh that
never disconnects" rather than "ssh in a tmux session".**
ET runs a small daemon (`etserver`) on the remote host listening
on TCP/2022 and a client (`et user@host`) on the local box; the
two negotiate a connection over plain TCP that the server keeps
alive even when the client vanishes for minutes / hours, and
when the client reconnects (from a new IP, on a new network)
the same shell session resumes exactly where it was — same
running command, same environment, same scrollback.

## Repo + version + license

- Repo: <https://github.com/MisterTea/EternalTerminal>
- Latest release: **`et-v6.2.11`** (2025-07-22)
- License: **Apache-2.0** —
  <https://github.com/MisterTea/EternalTerminal/blob/master/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `master`
- Language: C++ (server + client), with auth delegated to the
  host's existing OpenSSH

## Install

```bash
# macOS
brew install MisterTea/et/et

# Debian / Ubuntu (PPA)
sudo add-apt-repository ppa:jgmath2000/et
sudo apt update && sudo apt install et

# Arch
yay -S eternalterminal

# On the remote host: install the same package, start the server
sudo systemctl enable --now et

# Connect (uses ~/.ssh/config — same Host blocks as ssh)
et user@host

# Non-default port (server listens on TCP/2022 by default)
et --port 2022 user@host

# Jump-host shape: SSH-tunnel ET through a bastion
et -jumphost bastion.example.com user@private-host

# Port-forwarding (same -L / -R semantics as ssh)
et -t "8080:localhost:8080" user@host

# tmux integration: auto-attach a tmux session that survives
# even an etserver restart
et -c "tmux new -A -s main" user@host
```

## Niche

The "**ssh session that survives the network**" slot.

Plain `ssh` is connection-oriented: drop the TCP and the shell
dies, the running command gets a `SIGHUP`, the scrollback is
gone. The standard workaround is `ssh user@host -t 'tmux new -A
-s main'` so the *server-side* shell survives in a tmux session
even when ssh dies — this works, and most of the long-haul
remote work in the world runs on it. But the workaround has
real costs: every reconnect is a fresh TCP handshake + key
exchange + tmux attach + scrollback redraw (3–5 seconds on a
laptop wake), `Ctrl-C` semantics route through tmux's
key-binding layer instead of straight to the foreground
process, and the local terminal's native scrollback is empty
because tmux owns the terminal.

`mosh` is the obvious peer and the orthogonal choice. Mosh
solves the same "network drops should not kill the shell"
problem with a UDP-based State-Synchronization Protocol that is
brilliant on high-latency / lossy links (the local-echo
predictive display is the killer feature) — but it has two
hard constraints that drive teams to ET instead: (1) it
requires UDP/60000-61000 open between client and server, which
many corporate / cloud-network firewalls block by default, and
(2) it has no native scrollback — `Ctrl-Shift-Up` is a
`tmux`-style scrollback proxy because the mosh client owns the
terminal. ET runs over plain TCP/2022 (firewall-friendly) and
preserves the local terminal's native scrollback (the local
emulator's `Cmd-K` / `Ctrl-Shift-K` clears history that ET
kept intact across the disconnect), at the cost of mosh's
predictive local echo on lossy links.

Useful for:

- **Long-running remote work over flaky Wi-Fi / 4G / VPN** —
  laptop suspend / resume, café-Wi-Fi flake, train tunnels do
  not kill the shell.
- **Roaming between networks** — same `et` session resumes when
  you walk from the office to the coffee shop, IP changes
  transparently.
- **Corporate networks where UDP is blocked** — `mosh`'s UDP
  port range is firewalled out, ET's TCP/2022 goes through.
- **Remote dev with native scrollback** — `Cmd-Shift-Up` in
  iTerm / Alacritty / Kitty / WezTerm sees the actual scroll
  buffer the way it would for a local shell, instead of a
  tmux-mediated proxy.
- **Pairs cleanly with `tmux`** — ET keeps the network alive,
  tmux keeps the *server-side* session alive across
  `etserver` restarts; the combination is the "actually
  un-killable remote shell" pattern.

## Why it matters

- **Roaming TCP session** — server keeps the shell alive
  through arbitrary client disconnects and resumes the same
  session on reconnect, including IP / network changes.
- **Native scrollback preserved** — the local terminal owns
  the scroll buffer, ET round-trips updates as native terminal
  output; `Cmd-K` / scroll-back / search-in-history all work
  the way they do for a local shell.
- **Direct `Ctrl-C` semantics** — keystrokes route to the
  foreground process the way ssh does, not through a
  multiplexer's key-binding layer.
- **Plain TCP/2022** — works through corporate firewalls / NAT
  / cloud security groups that block mosh's UDP port range;
  the same firewall posture as `ssh -p 2022`.
- **Reuses OpenSSH for auth** — ET tunnels the initial auth
  through the host's `sshd` and reads `~/.ssh/config` for
  Host / IdentityFile / ProxyCommand, so existing ssh-key /
  agent / cert workflows just work; no separate user database.
- **Jump-host support** — `-jumphost` for a bastion, `-t` for
  port-forwarding, the flag set is sshish enough to drop into
  existing scripts.
- **Active maintenance, slower cadence** — `et-v6.2.11`
  (2025-07-22) is the most recent tagged release at snapshot
  time; the project ships a few releases a year, the
  protocol has been stable for years, and the codebase is
  small enough to audit (the trust boundary is a few thousand
  lines of C++ plus the system OpenSSH).
- **Honest scope** — ET is not a tmux replacement; the
  *server-side* shell still dies if `etserver` itself
  restarts (e.g. host reboot, OOM kill). Pair with `tmux` /
  `screen` / `zellij` for a truly indestructible session
  across both client and server failure. ET is also not
  better than mosh on a 10% packet-loss satellite link —
  mosh's predictive echo is real on those, and if your
  network is "high latency, high loss, UDP allowed", pick
  mosh.
- **vs `mosh`** — mosh is UDP-based with predictive local
  echo (better on lossy links, worse with corporate
  firewalls, no native scrollback); ET is TCP-based with
  native scrollback (firewall-friendly, no predictive echo).
  Pick mosh for satellite / cellular / hostile-network
  shapes; pick ET for corporate / cloud / native-scrollback
  shapes.
- **vs `tmux` over `ssh`** — that pattern keeps the
  *server-side* session alive but every reconnect is a fresh
  ssh handshake + tmux attach (multi-second delay) and the
  local scrollback is empty; ET makes the *connection*
  itself roaming so reconnect is sub-second and scrollback
  survives.
- **vs `autossh`** — autossh restarts dropped ssh connections
  but the underlying shell still dies (autossh + tmux is the
  workaround, with the same disadvantages as plain
  `ssh + tmux`); ET makes the shell itself survive.
- **Apache-2.0** — permissive; bundling `et` in a commercial
  product or an internal distribution is unencumbered with
  attribution.
