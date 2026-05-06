# iroh

> **iroh** — a modular, peer-to-peer networking stack in Rust that
> replaces "dial an IP and port" with "dial a public key" and
> handles NAT traversal, hole-punching, and an optional TURN-style
> relay automatically. Pinned to **v0.98.2**, dual Apache-2.0 / MIT —
> license file:
> [LICENSE-APACHE](https://github.com/n0-computer/iroh/blob/main/LICENSE-APACHE)
> (also
> [LICENSE-MIT](https://github.com/n0-computer/iroh/blob/main/LICENSE-MIT)).

Source: <https://github.com/n0-computer/iroh>

## TL;DR

Two endpoints generate keypairs, exchange the 32-byte public key out
of band (QR code, Slack DM, paste), and then `iroh` gives them a
QUIC stream — full-duplex, encrypted, authenticated by those keys —
between the two processes regardless of NAT topology, without
either side running on a public IP and without operating any
infrastructure of their own. Under the hood, `iroh` does the
hole-punching dance over a discovery network of relay servers (the
project runs a free-tier set, and you can `iroh relay` your own),
falls back to relayed traffic when direct paths fail, and exposes
the result as a regular `quinn` (QUIC) connection your application
code reads / writes bytes on.

The stack is shipped in three layers, each usable on its own:

- **`iroh`** (the crate + CLI) — the QUIC-over-public-keys
  transport. The CLI exposes `iroh node`, `iroh relay`,
  `iroh-dns-server`, plus diagnostic verbs (`iroh ping <node-id>`,
  `iroh doctor` for NAT / firewall reporting).
- **`iroh-blobs`** — content-addressed BLAKE3 blob transfer over an
  iroh connection (the "send a 4 GB file across the room without
  uploading it to the cloud first" verb).
- **`iroh-gossip`**, **`iroh-docs`** — pubsub and CRDT-replicated
  key-value documents on top of the same transport.

## Install

```bash
# Cargo
cargo install iroh-cli --locked

# Pre-built binaries from releases
# https://github.com/n0-computer/iroh/releases/tag/v0.98.2

# Homebrew (community tap)
brew install n0-computer/iroh/iroh
```

## Example commands

```bash
# Start an iroh node and print its node id (the public key)
iroh start

# Show NAT / firewall / relay diagnostics
iroh doctor

# Probe reachability of a remote node id
iroh ping bafkr4ig...

# Run your own relay (TURN-style fallback) on :3478 / :3479
iroh relay --dev
```

In a Rust program (the more common usage):

```rust
let endpoint = iroh::Endpoint::builder().bind().await?;
let conn = endpoint.connect(remote_node_id, b"my-app").await?;
let (mut send, mut recv) = conn.open_bi().await?;
send.write_all(b"hello").await?;
```

## Niche it occupies

**Public-key-addressed, NAT-traversing P2P transport** — different
niche from the rest of the catalog. Closest neighbours:

- [`croc`](../croc/) / [`magic-wormhole`](../magic-wormhole/) /
  [`piknik`](../piknik/) / [`ffsend`](../ffsend/) — *one-shot file
  transfers*. Pick those when the verb is "send this file once".
  Pick `iroh` when you want a *persistent connection* (RPC, sync,
  remote shell, app-to-app messaging) addressed by key not by IP.
- [`tailscale`](../tailscale/) / [`headscale`](../headscale/) /
  [`nebula`](../nebula/) / [`wireguard`](../wireguard/) — *overlay
  VPNs* that give every node an IP on a virtual network. Pick those
  when applications expect IP semantics. Pick `iroh` when the
  application *itself* speaks public keys (no virtual NIC, no kernel
  module, no admin coordination — keys are the address).
- [`syncthing`](../syncthing/) — *file-tree replication*. Pick
  syncthing for that one verb. Pick `iroh-docs` / `iroh-blobs` when
  you want the same NAT-traversal + BLAKE3 transfer story but
  embedded into your own Rust application instead of a standalone
  daemon.

Pairs cleanly with [`age`](../age/) (encrypt at rest before sending
over the iroh stream — defence in depth on top of QUIC's transport
encryption) and with [`quickwit`](../quickwit/) /
[`rqlite`](../rqlite/) (replicate operational data over an iroh mesh
without exposing a public socket).

## Citation

- Repo: <https://github.com/n0-computer/iroh>
- Latest release: **v0.98.2**
- License: **Apache-2.0** (dual-licensed Apache-2.0 OR MIT)
- License file: [LICENSE-APACHE](https://github.com/n0-computer/iroh/blob/main/LICENSE-APACHE)
