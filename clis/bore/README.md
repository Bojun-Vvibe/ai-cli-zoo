# bore

> **A simple CLI tool for making tunnels to localhost** — Rust
> client + server that exposes a local TCP port on a remote
> host's public address, with no account, no config file, and
> no protocol-level inspection (raw TCP, not HTTP-only).
> Pinned to **v0.6.0**
> ([LICENSE](https://github.com/ekzhang/bore/blob/main/LICENSE),
> MIT).

Source: <https://github.com/ekzhang/bore>

Category: networking / tunneling (localhost exposure)

## TL;DR

`bore local 8000 --to bore.pub` reverse-proxies your local
port 8000 to a randomly-assigned port on the public
`bore.pub` server (run by the author, free) and prints the
URL. Self-hosting is the point: `bore server` on any VPS
gives you a private tunnel endpoint with optional `--secret`
shared-key auth so only your laptops can connect. Single
~2 MB static binary; control protocol is a tiny custom JSON
handshake over TCP, not HTTP, so it tunnels SSH, Postgres,
WebSocket, anything stream-oriented.

## Install

```bash
# Homebrew
brew install bore-cli

# Cargo
cargo install --locked bore-cli

# Docker
docker run -it --rm --init --network host ekzhang/bore local 8000 --to bore.pub
```

## Why it sits in the zoo

The catalog already covers the HTTP-tunnel side (`ngrok`,
`localtunnel`, `cloudflared`); `bore` is the **raw-TCP, BYO-server,
no-account** alternative. Its 1-flag self-host story
(`bore server --min-port 1024 --secret $TOKEN`) is what
distinguishes it from frp / inlets / chisel for the simple
"laptop demo to client over coffee" use case.
