# dufs

> **Single-binary HTTP file server with a built-in web UI, WebDAV
> mount support, upload / delete / search, basic-auth ACLs per path,
> resumable uploads, and TLS — `dufs .` exposes the current directory
> on `:5000` with no config file, no Node runtime, and no Docker.**
> Written in Rust, ~8 MB binary, runs on Linux / macOS / Windows /
> FreeBSD / Android (Termux). Niche tag: **network / file-sharing**.
> Pinned to **v0.45.0**
> ([LICENSE-MIT](https://github.com/sigoden/dufs/blob/main/LICENSE-MIT)
> / [LICENSE-APACHE](https://github.com/sigoden/dufs/blob/main/LICENSE-APACHE),
> dual MIT / Apache-2.0).

Source: <https://github.com/sigoden/dufs>

## TL;DR

`dufs .` answers `GET` / `PUT` / `DELETE` / `PROPFIND` / `MKCOL`
on the current directory — meaning you can `curl -T file.bin
http://host:5000/` *or* mount the same URL from Finder / Explorer /
`davfs2` as a network drive. The web UI does folder browsing,
multi-select download as zip, drag-and-drop upload with progress,
in-browser preview for text / images / PDFs / video, and a search
box that walks the tree. Per-path auth (`-a /admin@user:pass:rw`,
`-a /pub@*:r`) lets one binary host a public read share and a
private read-write share without a reverse proxy. TLS is
`--tls-cert / --tls-key` flags away.

## Install

```bash
# Homebrew (macOS / Linux)
brew install dufs

# Cargo
cargo install dufs

# Direct binary (Linux x86_64)
curl -L https://github.com/sigoden/dufs/releases/download/v0.45.0/dufs-v0.45.0-x86_64-unknown-linux-musl.tar.gz \
  | tar -xz && sudo mv dufs /usr/local/bin/
```

## Example

```bash
# Serve the current directory read-only on :5000
dufs .

# Read-write with auth, allow uploads + deletes + symlinks, on :8080
dufs -A -a 'admin:s3cret@/:rw' -a '*:r@/' --bind 0.0.0.0 -p 8080 ./share

# Serve over HTTPS with a Let's Encrypt cert and enable WebDAV mount
dufs --tls-cert fullchain.pem --tls-key privkey.pem --enable-cors ./public
```

## When to use

- You need to *temporarily* expose a directory to a teammate, a
  device on the LAN, or a CI runner without standing up nginx,
  Caddy, or `python -m http.server` (which lacks uploads, auth,
  and WebDAV).
- You want a one-binary WebDAV target for `rclone`, `davfs2`, iOS
  Files.app, or a self-hosted Joplin / Obsidian sync.
- You're building a "drop a binary on a NAS / Raspberry Pi / Termux
  phone and have a file server" setup and don't want a Node /
  Python / Go runtime in the dependency chain.

## When NOT to use

- You need a real *cloud-storage* surface — multi-user accounts,
  shared albums, calendars, contacts, mobile sync apps — use
  Nextcloud / Seafile / Filebrowser; dufs is intentionally a single
  binary, not a platform.
- You need S3-compatible object semantics — use MinIO or Garage.
- You want fine-grained per-file ACLs, audit logs, or LDAP / SSO —
  dufs's auth model is path-prefix + basic-auth and stops there.
