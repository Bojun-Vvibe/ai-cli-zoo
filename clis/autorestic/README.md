# autorestic

> **Declarative wrapper around `restic`: one YAML file (`.autorestic.yml`)
> describes every backup *location* (a directory or volume to back up),
> every *backend* (S3, B2, Backblaze, local disk, SFTP, Rclone target),
> and the cron schedule + prune policy for each — then `autorestic
> backup -a` and `autorestic cron` do the rest.** Pinned to **v1.8.3**,
> Apache-2.0
> ([LICENSE](https://github.com/cupcakearmy/autorestic/blob/master/LICENSE)).

- **Repo:** https://github.com/cupcakearmy/autorestic
- **Latest version:** v1.8.3 (released 2025-08-28)
- **License:** Apache-2.0 (`LICENSE` at repo root)
- **Category:** `backup` / `restic-wrapper` / `homelab` / `disaster-recovery`
- **Language:** Go

## What it does

`restic` is the de-facto modern encrypted-deduplicating backup tool,
but its CLI is procedural: every invocation needs `RESTIC_REPOSITORY`,
`RESTIC_PASSWORD`, the source path, the include/exclude flags, the
prune policy. In a homelab or single-tenant server with a dozen
things worth backing up (Postgres dumps, a media library, `/etc`,
container volumes, a Nextcloud data dir) and three places to back
them up to (an S3 bucket, a Backblaze B2 bucket, a USB disk), you end
up writing a lot of fragile bash. `autorestic` replaces that bash
with one declarative YAML: each *location* lists its `from`,
`to` (one or more backend names), `cron` schedule, optional
pre/post hooks (e.g. `pg_dump` before backup), and forget policy
(`keep-daily 7 keep-weekly 4 keep-monthly 12`). `autorestic backup
-a` runs every location to every backend in parallel; `autorestic
forget -a --prune` enforces retention; `autorestic cron` is a single
long-running process suitable for systemd. The repository password
is generated and stored once in `~/.autorestic.yml`, not pasted into
shell history. Restore is symmetric: `autorestic restore <location>
--from <backend> --to /tmp/restore`.

## Install

```bash
# Official install script (downloads the right binary for your arch)
wget -qO- https://raw.githubusercontent.com/cupcakearmy/autorestic/master/install.sh | sudo bash

# macOS via Homebrew
brew install autorestic

# Or grab a release binary directly
curl -L https://github.com/cupcakearmy/autorestic/releases/download/v1.8.3/autorestic_linux_amd64 \
  -o /usr/local/bin/autorestic && chmod +x /usr/local/bin/autorestic
```

## Examples

```yaml
# .autorestic.yml — the entire backup plan for a small server
version: 2

locations:
  postgres:
    from: /var/lib/postgresql
    to: [b2-offsite, usb-local]
    hooks:
      before: ["pg_dumpall -U postgres > /var/lib/postgresql/dump.sql"]
    cron: "0 3 * * *"
    forget:
      keep-daily: 7
      keep-weekly: 4
      keep-monthly: 12

  nextcloud-data:
    from: /srv/nextcloud/data
    to: b2-offsite
    cron: "0 4 * * *"

backends:
  b2-offsite:
    type: b2
    path: my-bucket:server-01
    key:
      B2_ACCOUNT_ID: ${B2_ACCOUNT_ID}
      B2_ACCOUNT_KEY: ${B2_ACCOUNT_KEY}
  usb-local:
    type: local
    path: /mnt/usb-backup
```

```bash
autorestic check                    # verify config + connectivity
autorestic backup -a                # back everything up everywhere now
autorestic forget -a --prune        # apply retention, free space
autorestic restore postgres --from b2-offsite --to /tmp/restore
autorestic cron                     # long-running scheduler (systemd-friendly)
```

## Why it matters in an AI-native workflow

The same agent loop that writes infrastructure code (Terraform,
Ansible, container Compose files) inevitably needs to *prove* the
infrastructure is recoverable, and "prove" means a tested restore
from a real off-host backup. Hand-rolled `restic` scripts are
exactly the kind of thing an LLM will produce that *looks* right
and silently never runs. `autorestic`'s declarative YAML collapses
the surface area to one reviewable file: an agent reading
`.autorestic.yml` knows the full backup posture in 30 lines, can
diff a proposed change ("add a new location for the new service")
without rewriting bash, and can ask `autorestic check` for a
machine-verifiable answer about whether every backend is actually
reachable. Pairs with [`restic`](../restic/) (the engine doing the
real work — autorestic is *only* the orchestrator), with
[`rclone`](../rclone/) (autorestic supports rclone-typed backends,
so you get backups to anything rclone can mount: Google Drive,
Dropbox, OneDrive-equivalent), and with [`croc`](../croc/) for
ad-hoc out-of-band restoration of a single file. Orthogonal to
snapshot tools like [`borgbackup`](../borgbackup/) which use a
different on-disk format and have no equivalent declarative
multi-backend wrapper of similar maturity.
