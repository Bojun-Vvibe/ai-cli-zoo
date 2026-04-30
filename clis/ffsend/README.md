# ffsend

> **Easily and securely share files from the command line** —
> a fully-featured Rust client for Send (the Firefox Send
> protocol), with end-to-end encryption, expiring links,
> download caps, and optional passwords. Pinned to **v0.2.77**
> ([LICENSE](https://github.com/timvisee/ffsend/blob/master/LICENSE),
> GPL-3.0).

Source: <https://github.com/timvisee/ffsend>

Category: networking / file-transfer

## TL;DR

`ffsend upload ./report.pdf` returns a single one-line URL
whose fragment carries the AES-GCM key, so the server (any
Send-compatible host — the original Mozilla service is gone,
but `send.vis.ee` and self-hosted instances run the protocol)
never sees plaintext. `--expiry-time`, `--download-limit`,
`--password`, and `--archive` (auto-tar a directory) make
one-off transfers between humans a single command. Companion
verbs: `download`, `info`, `delete`, `parameters`, `password`,
`history`.

## Install

```bash
# Homebrew
brew install ffsend

# Arch:    pacman -S ffsend
# Nix:     nix-env -iA nixpkgs.ffsend
# Snap:    snap install ffsend

# Cargo
cargo install --locked ffsend
```

## Why it sits in the zoo

Compared to `croc` (PAKE-based peer-to-peer) and
`magic-wormhole` (also PAKE), `ffsend` is the *server-mediated*
end of the spectrum: receiver doesn't need any tool installed,
just a browser. The download-limit + expiry + password trio
make it the right pick for "send a build artifact to a
non-technical reviewer and have it self-destruct in 24 h".
