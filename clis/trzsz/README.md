# trzsz

> **Drop-in zmodem-style file transfer over an existing SSH /
> tmux session**, but with a modern UX: drag-and-drop upload
> from a supported terminal emulator (iTerm2, tabby, electerm,
> WindTerm, trzsz-ssh), text-mode progress bar, breakpoint
> resume, transparent compression, OSC52 clipboard
> integration, and a Go binary that wraps an arbitrary login
> command instead of needing patched `rz` / `sz` on the
> server. Pinned to **v1.2.0** (released 2026-01-17,
> [LICENSE](https://github.com/trzsz/trzsz-go/blob/v1.2.0/LICENSE),
> MIT).

Source: <https://github.com/trzsz/trzsz-go>

## TL;DR

You're SSH'd into a jump host, three hops deep, no shared
filesystem, no S3 bucket lying around, and you need to push
one config file up or pull one core dump down. `scp` requires
a fresh second connection (often blocked by the bastion);
`rsync` over the existing session needs a tunnel; `lrzsz` /
`zmodem` works but is painful inside `tmux` and blind in modern
terminals. `trzsz` keeps the existing TTY: you run `trz` on
the remote to upload or `tsz file` to download, and the local
`trzsz` binary that wraps your shell's `ssh` invocation
intercepts the protocol and does the transfer over the same
socket. Inside `tmux` it works without a side-channel; in a
supporting terminal you can drag a file onto the window
instead of typing a path.

## Install

```bash
# macOS (Homebrew)
brew install trzsz

# Linux (deb / rpm / APK / Arch)
# Releases page: https://github.com/trzsz/trzsz-go/releases/tag/v1.2.0
sudo dpkg -i trzsz_1.2.0_linux_amd64.deb

# Go install (cross-platform)
go install github.com/trzsz/trzsz-go/cmd/trzsz@v1.2.0
go install github.com/trzsz/trzsz-go/cmd/trz@v1.2.0
go install github.com/trzsz/trzsz-go/cmd/tsz@v1.2.0

# Verify
trzsz --version          # trzsz 1.2.0
```

## License

MIT — see
[LICENSE](https://github.com/trzsz/trzsz-go/blob/v1.2.0/LICENSE).
Permissive: bundle freely in proprietary toolchains; no
copyleft obligation on dotfiles or wrapper scripts.

## Common invocations

```bash
# Wrap your existing ssh / mosh / etc. with trzsz on the local side
trzsz ssh user@bastion
# Inside the remote shell, upload from local:
trz                       # picks files via local TUI / drag-drop
trz ./incoming            # save into ./incoming directory

# Download a file from the remote to local cwd
tsz /var/log/app.crash    # progress bar in the remote TTY

# Resume a previously interrupted transfer (default since v1.1.4)
tsz -y bigfile.tar.gz     # auto-overwrite, breakpoint resume

# Transfer a directory with on-the-fly compression
trz -d -z ./build         # -d directory, -z compress

# Inside tmux — works without zmodem side-channel since v1.1.0
tmux a -t work
tsz dump.bin              # transfer continues across tmux re-attach

# Background transfer (don't block the prompt)
tsz -B logs/*.log
```

## Why use it

- **No second connection.** Reuses the SSH session you
  already authenticated. Critical when the bastion firewall
  permits one TCP connection but you still need to move a
  file.
- **`tmux`-aware.** The protocol negotiates a tmux-friendly
  framing in v1.1.0+, so a transfer started in a detached
  pane survives reattach without a separate side channel.
- **Drag-and-drop in supported terminals.** iTerm2, tabby,
  electerm, WindTerm, and `trzsz-ssh` recognise the protocol
  and let you drop a file onto the window — the local trzsz
  binary then streams it over the existing TTY.
- **Single static binaries.** `trzsz` (the wrapper),
  `trz` (upload helper for the remote shell), and `tsz`
  (download helper) are three Go binaries with no runtime
  dependency. Drop them in `/usr/local/bin` on either side.

## Vs Already Cataloged

- **Vs `scp` / `rsync` / `sftp`:** all need a separate
  connection, separate auth, and often a separate firewall
  rule. `trzsz` rides the existing SSH stream.
- **Vs [`magic-wormhole`](../magic-wormhole/):** wormhole is
  a peer-to-peer transfer with a shared rendezvous server
  and a one-time PAKE code — beautiful for arbitrary
  laptop-to-laptop, but useless on a bastion you can't
  reach a STUN server from. `trzsz` is the right answer
  inside firewalled corporate networks where the SSH
  socket is the only egress.
- **Vs [`croc`](../croc/):** same rendezvous-server model
  as wormhole, same constraint. Pick `croc` for ad-hoc
  laptop sharing, `trzsz` for "I'm already inside an SSH
  session and can't open a new socket".
- **Vs [`piknik`](../piknik/):** `piknik` is a network
  clipboard with a long-lived server. Different shape: it
  syncs short payloads via a server you run; `trzsz` moves
  arbitrary files over a session you already have.
- **Vs [`ffsend`](../ffsend/):** `ffsend` uploads to a
  Send-protocol server (Mozilla Send compatible) and gives
  you a URL. `trzsz` is hop-direct, no third party.

## Caveats

- **Helper binaries on both sides.** The remote needs `trz`
  and `tsz` on `$PATH`; the local needs `trzsz` wrapping
  your `ssh`. Often shipped via `brew install trzsz` /
  package manager; in a locked-down environment you may
  need to `scp` the helpers in once.
- **Terminal emulator support varies.** Drag-and-drop only
  works in iTerm2, tabby, electerm, WindTerm, and
  `trzsz-ssh`. Other terminals fall back to the TUI file
  picker — still works, just no drag.
- **Protocol over a TTY, not a binary stream.** Throughput
  caps below pure `scp` over a clean socket; the win is
  reachability, not raw speed. For multi-GB transfers on a
  reachable host, `rsync -P` is still faster.
- **Protocol-level compression has bookkeeping cost.**
  `-z` (auto-compression since v1.1.4) helps for text /
  source / logs; for already-compressed payloads
  (videos, `.tar.zst`, `.parquet`) leave it off.
