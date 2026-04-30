# litestream

- **Repo:** https://github.com/benbjohnson/litestream
- **Version:** v0.5.11
- **License:** [LICENSE](https://github.com/benbjohnson/litestream/blob/main/LICENSE) (Apache-2.0)
- **Category:** Streaming, point-in-time replication for SQLite

## What it is

`litestream` is a small Go daemon (and CLI) that turns a single-node SQLite
database into something you can actually run a service on top of. It tails
SQLite's WAL, ships incremental segments to object storage (S3, GCS, Azure
Blob, SFTP, ABS, or a local filesystem), and lets you restore the database
to any second within the retention window. There is no fork of SQLite, no
custom VFS, and no application library to integrate — your app keeps using
plain `sqlite3` and Litestream rides alongside the file. The same binary
also runs the **restore** path, which is what you call from a fresh VM or
container at boot to rehydrate the DB before the app opens it.

## Install

```
brew install benbjohnson/litestream/litestream                    # macOS / Linuxbrew
# or grab the binary from https://github.com/benbjohnson/litestream/releases
litestream version
```

## Basic usage

```
# /etc/litestream.yml
# dbs:
#   - path: /var/lib/app/app.db
#     replicas:
#       - url: s3://my-bucket/app-db

litestream replicate                                              # foreground daemon
litestream replicate -config /etc/litestream.yml                  # explicit config
litestream snapshots s3://my-bucket/app-db                        # list known snapshots
litestream restore -o /var/lib/app/app.db s3://my-bucket/app-db   # rehydrate from object store
litestream restore -timestamp 2026-04-30T12:00:00Z \
  -o /tmp/app.db s3://my-bucket/app-db                            # PITR to a moment
```

## When to use it

- You are running a **single-node service backed by SQLite** (an internal
  tool, an edge worker, a small SaaS) and want durable backups + PITR
  without standing up Postgres just for the DR story.
- You want **continuous, asynchronous replication to cheap object storage**
  with second-level RPO, not a nightly `sqlite3 .backup` cron.
- You deploy on **ephemeral compute** (Fly.io, Render, container per VM)
  and need a "boot script restores DB from S3, then start the app" pattern
  that just works.
- Skip it when you need **multi-writer / horizontally scaled SQL**; the
  sister project `rqlite` or a real client/server database is the right
  tool, not Litestream.
