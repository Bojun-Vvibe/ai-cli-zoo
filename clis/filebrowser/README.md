# filebrowser

> **A single-binary Go web UI for browsing, uploading,
> downloading, sharing, and editing files inside one
> directory** — runs as `filebrowser -r /path/to/serve` and
> exposes a real multi-user web app on `:8080`: tree
> navigation, drag-and-drop uploads, in-browser preview for
> images / videos / PDFs / audio / Markdown / source, inline
> file editing, archive extraction, per-user JWT auth with
> per-folder permissions, and time-limited public share links.
> Pinned to **v2.63.3** (SPDX: `Apache-2.0`,
> [LICENSE](https://github.com/filebrowser/filebrowser/blob/master/LICENSE)).

Source: <https://github.com/filebrowser/filebrowser>

## TL;DR

`filebrowser` is the right reach when [`miniserve`](../miniserve/)
or [`dufs`](../dufs/) is too read-only and a NAS / Nextcloud /
Seafile is too much. One binary (`~30 MB`), one SQLite database
for users + permissions + share links, one config file (or pure
flags), and the UI is a real SPA with auth — not a directory
listing with a download button. Multi-user, multi-folder
permissions, and the share-link generator are the features that
push it over the line for "I want a self-hosted Dropbox-shaped
folder for the team."

## Install

```bash
# Homebrew
brew install filebrowser/tap/filebrowser

# Single-shell installer (Linux / macOS)
curl -fsSL https://raw.githubusercontent.com/filebrowser/get/master/get.sh | bash

# Docker (the production-default deploy shape)
docker run -d \
  -v /srv/files:/srv \
  -v /srv/filebrowser.db:/database.db \
  -v /srv/filebrowser.json:/.filebrowser.json \
  -p 8080:80 \
  filebrowser/filebrowser:v2.63.3

# From a release tarball
curl -L -o filebrowser.tar.gz \
  https://github.com/filebrowser/filebrowser/releases/download/v2.63.3/linux-amd64-filebrowser.tar.gz
tar xzf filebrowser.tar.gz && sudo mv filebrowser /usr/local/bin/

# Verify
filebrowser version
```

## Usage

```bash
# 1. Quickest possible: serve cwd, default admin/admin login,
#    listen on :8080
filebrowser -r .

# 2. Production shape: dedicated DB, dedicated root, bound to
#    localhost (Caddy / nginx terminates TLS)
filebrowser \
  --address 127.0.0.1 \
  --port 8080 \
  --root /srv/files \
  --database /var/lib/filebrowser/filebrowser.db \
  --baseurl /files

# 3. First-time setup: create a non-default admin
filebrowser users add alice 'a-real-password' --perm.admin

# 4. Add a non-admin user with read-only access to one subfolder
filebrowser users add bob 'pwd' --scope /srv/files/shared --perm.create=false --perm.delete=false --perm.modify=false

# 5. Reset a forgotten password
filebrowser users update alice --password 'new-password'

# 6. Create a public share link from inside the UI:
#    Right-click a file → Share → set duration (24h / 7d / forever)
#    → set optional password → copy URL
#    Anyone with the URL gets a download (or upload, if enabled)
#    without an account.

# 7. Run as a systemd unit (typical deploy)
sudo tee /etc/systemd/system/filebrowser.service <<'EOF'
[Unit]
Description=File Browser
After=network.target

[Service]
ExecStart=/usr/local/bin/filebrowser -d /var/lib/filebrowser/filebrowser.db -r /srv/files
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now filebrowser
```

## Why it matters

The "I want a folder on my server, accessible by humans, over a
browser, with auth" shape is otherwise solved either too lightly
(a static-server like miniserve / dufs gives you read access and
maybe upload, but no users, no permissions, no share links) or
too heavily (Nextcloud / Seafile / ownCloud bring a PHP runtime,
a MySQL/Postgres database, a calendar, a contacts app, a
collaborative-editing module, a webmail, a market for plugins —
hundreds of MB of dependencies for what was supposed to be "a
folder over HTTPS"). filebrowser sits in the gap: real auth,
real permissions, real share links, real preview / inline edit,
**no** PHP / no Postgres / no app market — one Go binary, one
SQLite database, one JSON config, deployable in a container in
20 seconds.

## Niche It Fills

**Multi-user web file browser with auth and shares, single
binary.** The space splits three ways: read-mostly directory
servers (miniserve, dufs, Caddy's `file_server` — fine, but no
users / no shares / no preview), self-hosted "folder as a
service" platforms (Nextcloud, Seafile, ownCloud — heavy, full
collaboration suites with PHP / DB requirements), and
filebrowser (one Go binary + SQLite). It is the canonical pick
when "give five people a web UI to a folder, with their own
logins and permissions, and let them share files outside without
creating accounts" is the actual requirement.

## Vs Already Cataloged

- **Vs [`miniserve`](../miniserve/):** miniserve is the
  one-shell-flag static server (`miniserve .`) with optional
  upload and HTTP-basic-auth. It has no user database, no
  per-folder permissions, no share-link generator, no inline
  preview, no editing. Pick miniserve for "send this folder to
  one colleague for ten minutes"; pick filebrowser for "host
  this folder for the team for a year, with separate logins and
  audit-able share links."
- **Vs [`dufs`](../dufs/):** dufs is the modern Rust evolution
  of miniserve — faster, with WebDAV support, optional auth,
  and a richer single-page UI for browsing. Still single-user
  in spirit (no user-management UI, no per-user permissions in
  a database). Pick dufs when WebDAV mounting from macOS Finder
  / Windows Explorer is the killer feature; pick filebrowser
  when web UI + multi-user + share links is the killer feature.
- **Vs [`soft-serve`](../soft-serve/):** different shape entirely
  — soft-serve is a self-hosted git server browseable over SSH /
  TUI / web. filebrowser is a generic file folder. Pick soft-serve
  for "host my git repos"; pick filebrowser for "host arbitrary
  files."
- **Vs Nextcloud / Seafile (not cataloged):** Nextcloud and
  Seafile are the heavyweight picks — collaborative editing,
  calendars, contacts, plugin marketplaces, mobile sync clients,
  group chat. filebrowser trades all of that for "the folder, a
  web UI, users, shares, done." Pick the heavyweights when you
  actually need calendar / contacts / sync clients; pick
  filebrowser when 90% of what you need from "self-hosted
  Dropbox" is just the file UI.

## Caveats

- **Not a sync client.** filebrowser is a *web UI* over a
  folder. It does not push or pull from desktop sync clients
  the way Nextcloud / Dropbox do. If you want a folder that
  auto-mirrors between two laptops, pair with
  [`syncthing`](../syncthing/) (peer-to-peer sync) and use
  filebrowser only for the web access layer on top of the same
  folder.
- **Default admin/admin on first run.** A fresh install with
  no `--noauth` flag and no `users add` step ships with
  `admin / admin` as the bootstrap credentials. Change the
  password immediately, or pre-create the admin via
  `filebrowser users add` before exposing the port. Do not
  expose the default credentials to the public internet for
  even one minute.
- **TLS is your responsibility.** filebrowser can serve TLS
  directly (`--cert` / `--key`), but the canonical deploy puts
  it behind [`caddy`](../caddy/) / nginx / Traefik for cert
  management, HTTP/2, and rate limiting. Direct exposure on
  `:80` / `:443` works but skips the auto-renew layer.
- **SQLite database is the source of truth for users and
  shares.** Back up `filebrowser.db` alongside the served
  folder; losing it loses every user account, every per-folder
  permission, and every active share link. The database is
  small (KB to a few MB) — `sqlite3 filebrowser.db .dump >
  backup.sql` is enough.
- **Inline editor is text-only.** filebrowser's "edit this
  file" pane is a Monaco / Ace web editor for plain text /
  source / Markdown — fine for `nginx.conf` tweaks and
  Markdown drafts, not a substitute for collaborative editing
  of `.docx` / `.xlsx`. Use Nextcloud + Collabora / OnlyOffice
  for that case.
- **Share-link revocation is manual.** Share links can have an
  expiry, but if you create one with no expiry and forget about
  it, it remains live until you delete it from the share-list
  UI. Audit the active-shares page periodically; consider
  always setting an expiry by default.
