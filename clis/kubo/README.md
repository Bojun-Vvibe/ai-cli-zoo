# kubo

> **The reference Go implementation of IPFS — one daemon
> (`ipfs daemon`) plus a verb-noun CLI (`ipfs add`, `ipfs cat`,
> `ipfs pin add`, `ipfs name publish`, `ipfs swarm peers`,
> `ipfs dag get`, `ipfs files cp`) that exposes the entire
> content-addressed file / DAG / pubsub / DHT / IPNS surface of
> the InterPlanetary File System over a local HTTP API on
> 127.0.0.1:5001 and a public libp2p swarm on 4001/tcp+udp.**
> Pinned to **v0.41.0** (released 2026-04-23),
> [LICENSE](https://github.com/ipfs/kubo/blob/master/LICENSE) /
> [LICENSE-MIT](https://github.com/ipfs/kubo/blob/master/LICENSE-MIT) /
> [LICENSE-APACHE](https://github.com/ipfs/kubo/blob/master/LICENSE-APACHE),
> dual MIT + Apache-2.0.

Source: <https://github.com/ipfs/kubo>

## TL;DR

`kubo` is the original and still the canonical IPFS node — a
single Go binary (`ipfs`) that runs as both a long-lived daemon
joining the public DHT swarm and a CLI client driving the
daemon's local HTTP API. Every artifact you put through it
(`ipfs add ./report.pdf`) is hashed into a content identifier
(CID, e.g. `bafybeigdyrzt5...`) that is the address — change
one byte and you get a different CID, fetch the same CID from
any peer in the world and you get bit-identical bytes. The
DAG layer (`ipfs dag put` / `ipfs dag get`) lets you build
arbitrary IPLD-shaped graphs (Merkle DAGs of JSON / CBOR /
protobuf nodes) and walk them by CID + path; the MFS layer
(`ipfs files`) gives you a mutable POSIX-shaped filesystem
view on top of the immutable DAG so a workflow looks like
`mkdir` / `cp` / `mv` instead of "compute new root CID by
hand". IPNS (`ipfs name publish <cid>`) signs a CID under your
node's libp2p identity so others can resolve `/ipns/<peer-id>`
to the latest CID without you handing them a new hash each
time. Pubsub (`ipfs pubsub sub <topic>`) and the DHT
(`ipfs dht findprovs <cid>`) are first-class CLI verbs, so
real applications (decentralised package mirrors,
content-addressed CI caches, p2p backup grids) can be wired
together from shell scripts before you reach for a library.

## Install

```bash
# Homebrew (macOS / Linux)
brew install ipfs

# Static binary from dist.ipfs.tech (linux / macOS / windows, amd64 / arm64)
curl -fsSL -o /tmp/kubo.tar.gz \
  https://github.com/ipfs/kubo/releases/download/v0.41.0/kubo_v0.41.0_darwin-arm64.tar.gz
tar -xzf /tmp/kubo.tar.gz -C /tmp
sudo /tmp/kubo/install.sh

# Docker (great for a pinned, sandboxed node)
docker run -d --name ipfs-host \
  -v $PWD/ipfs-data:/data/ipfs \
  -p 4001:4001 -p 4001:4001/udp \
  -p 127.0.0.1:5001:5001 -p 127.0.0.1:8080:8080 \
  ipfs/kubo:v0.41.0

# Verify
ipfs version    # ipfs version 0.41.0

# First-time init
ipfs init                         # writes ~/.ipfs/, generates peer key
ipfs daemon &                     # joins the public swarm
ipfs id                           # prints your peer ID + multiaddrs
```

## One Concrete Example

```bash
# Publish a directory as a content-addressed site, mutate it,
# republish under a stable IPNS name, and fetch it from a
# different machine without ever talking to a central server.

# 1. Add a directory recursively — every file gets its own CID,
#    the dir gets a parent CID
CID=$(ipfs add -Q -r ./site/)
echo "site root: $CID"
# bafybeigdyrzt5sfbtgz... (32-byte sha256 base32-encoded)

# 2. Pin it locally so GC won't drop it
ipfs pin add $CID

# 3. Browse it via the local gateway (read-only HTTP)
curl -s http://127.0.0.1:8080/ipfs/$CID/index.html | head

# 4. Publish the CID under your node's stable IPNS name
ipfs name publish /ipfs/$CID
# Published to k51qzi5uqu5dk...: /ipfs/bafybeigdyrzt5...

# 5. Mutate the site, republish — the IPNS name now resolves to
#    the new CID; consumers don't have to learn a new hash
echo "<h1>v2</h1>" > ./site/index.html
NEW_CID=$(ipfs add -Q -r ./site/)
ipfs name publish /ipfs/$NEW_CID

# 6. From any other kubo node on the planet
ipfs name resolve k51qzi5uqu5dk...        # → /ipfs/bafybeih...new...
ipfs cat /ipns/k51qzi5uqu5dk.../index.html

# 7. MFS view — treat the immutable DAG like a mutable filesystem
ipfs files mkdir -p /myproj
ipfs files cp /ipfs/$NEW_CID /myproj/site
ipfs files ls /myproj
ipfs files stat /myproj/site
```

For programmatic use the same daemon exposes an HTTP API on
`127.0.0.1:5001/api/v0/` (POST-only, multipart) — every CLI
verb is an endpoint, so any language with an HTTP client is a
first-class kubo client.

## License

Dual-licensed [MIT](https://github.com/ipfs/kubo/blob/master/LICENSE-MIT)
+ [Apache-2.0](https://github.com/ipfs/kubo/blob/master/LICENSE-APACHE);
top-level [LICENSE](https://github.com/ipfs/kubo/blob/master/LICENSE)
states "this code is dual-licensed under MIT + Apache-2.0".
SPDX `MIT OR Apache-2.0`.

## Niche / positioning

The **canonical CLI for the public IPFS network** —
content-addressed, peer-to-peer, no central server, every
artifact is its own permanent address. Pick `kubo` over a
hand-rolled HTTP+sha256 cache when you want global
deduplication (the same byte sequence anywhere has the same
CID), automatic peer discovery (DHT), free CDN-like behaviour
(any peer that has the CID can serve it), and signed mutable
pointers (IPNS) without standing up Cloudflare / S3 / a
signing service. Pick over [`rclone`](../rclone/) /
[`s5cmd`](../s5cmd/) / [`s3cmd`](../s3cmd/) when the address
should survive the storage backend (move bytes between nodes,
the CID stays valid). Pick over [`borgbackup`](../borgbackup/)
/ [`duplicacy`](../duplicacy/) when you want shared / public
distribution and not encrypted private snapshots — kubo's DAG
is unencrypted by default and globally addressable, the
opposite trust model. Skip when the workload is
"private-bucket S3 with an ACL" (use the cloud's CLI), when
you need strong delete semantics (content-addressed networks
have no "unpublish — please forget"), or when bandwidth /
disk overhead from joining the public DHT is unacceptable
(run `ipfs config Routing.Type none` for a private node, or
use a different stack entirely).
