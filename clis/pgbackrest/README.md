# pgbackrest

- **Repo:** https://github.com/pgbackrest/pgbackrest
- **Version:** release/2.58.0 (2026-01-19)
- **License:** MIT ([LICENSE](https://github.com/pgbackrest/pgbackrest/blob/main/LICENSE))
- **Language:** C (single static binary, depends on libpq + OpenSSL + libxml2 + libssh2 + lz4 + zstd + libbz2)
- **Install:** `brew install pgbackrest` · `apt install pgbackrest` (Debian / Ubuntu, also packaged in PGDG repos) · `dnf install pgbackrest` (RHEL / Rocky / Alma via PGDG) · pre-built binaries shipped in the PostgreSQL community APT / YUM repositories tracked by every supported PG version

## What it does

`pgbackrest` is a **PostgreSQL-aware backup and restore engine** designed for production databases that outgrow `pg_dump` and `pg_basebackup` — typically anything with a recovery-time / recovery-point objective measured in minutes, anything in the hundreds-of-GB to multi-TB range, anything that needs point-in-time recovery (PITR), and anything that wants its backups stored in object storage rather than on a backup server's local disk. The CLI surface is small and stable: `pgbackrest --stanza=<name> stanza-create` initializes the metadata for a new database (a "stanza" is one PG cluster from pgbackrest's perspective), `pgbackrest --stanza=<name> backup --type=full|diff|incr` takes a full / differential / incremental backup, `pgbackrest --stanza=<name> info` prints the human-readable + JSON-able catalog of every backup with size / duration / WAL range / encryption status, `pgbackrest --stanza=<name> restore` restores the latest backup (or `--type=time --target='2026-04-30 12:00:00'` for PITR, or `--type=name --target='before_migration'` for named-restore-point), `pgbackrest --stanza=<name> archive-push %p` is the `archive_command` PostgreSQL invokes for every WAL segment, and `pgbackrest --stanza=<name> expire` enforces retention. Behind that small surface is the engineering that distinguishes it: parallel backup and restore (`process-max=8` runs eight worker processes pushing files concurrently to the repository, saturating gigabit links and NVMe arrays), block-level incremental backup (`block-incr=y` checksums file blocks rather than whole files so a 1 TB database with 50 GB of changes produces a 50 GB incremental, not a 1 TB one), per-file Zstandard / LZ4 / gzip / bzip2 compression chosen per backup (`compress-type=zst --compress-level=3` is the modern default), AES-256-CBC repository-side encryption with the key supplied at command time (`repo1-cipher-type=aes-256-cbc`), three independent repositories simultaneously (`repo1-*` local NVMe + `repo2-*` S3 + `repo3-*` SFTP — pgbackrest pushes to all three on every `backup`), native S3 / GCS / Azure Blob / SFTP storage drivers (no `aws s3 cp` shim), automatic WAL archiving with async parallel push (`archive-async=y` lets PostgreSQL's `archive_command` return in microseconds while pgbackrest streams WAL to the repo in the background), restore-time delta mode (`--delta` only restores the file blocks that differ from what is already on disk, so a "rewind a 5 TB replica to 4 hours ago" operation moves only the diff), and standby-side backup (`backup-standby=y` reads pages from a hot standby to offload I/O from the primary). Verify mode (`pgbackrest verify`) re-checksums every file in the repository against the manifest to catch silent corruption before the disaster.

## When to pick it / when not to

Pick `pgbackrest` when the PostgreSQL database is a tier-1 production system: a SaaS product's primary OLTP store, a financial ledger, a clinical / regulated dataset, a multi-TB analytics warehouse, an ops-on-call PG cluster behind a payment processor. Concrete cases: a 4 TB PG 18 cluster on bare metal where `pg_basebackup` takes 6 hours and burns network — pgbackrest's parallel block-incremental backups complete in 10 minutes after the initial full and the restore is also parallel; a fleet of 60 PG instances across regions all backing up to per-region S3 buckets with a single pgbackrest config templated by Ansible / Terraform; a regulated environment that requires AES-256 encryption at rest in the backup repository plus an audit trail per backup (the JSON `info` output drops directly into compliance evidence); a "rewind production to 12:34:56 yesterday" PITR drill that the team practices quarterly because `--type=time --target=...` is documented and reliable; a high-availability pair where backups are taken from the standby with `backup-standby=y` to keep the primary's I/O budget for queries. Pair with [`patroni`](../patroni/) for the HA / leader-election layer (Patroni's docs explicitly recommend pgbackrest for the backup tier and call its `archive_command` directly); pair with [`pgbouncer`](../pgbouncer/) for the connection-pool layer; pair with [`pg_partman`](https://github.com/pgpartman/pg_partman) for partition lifecycle. The `kubegres` / `cnpg` / `zalando-postgres-operator` Kubernetes operators all integrate pgbackrest as the canonical backup driver, so adopting it on bare metal is also a step toward a future cluster-on-Kubernetes migration.

Skip `pgbackrest` for a single hobby database where `pg_dump | gzip > daily.sql.gz` plus a cron job is enough — pgbackrest's value is in parallelism, incremental backup, PITR, and object-storage integration, all of which are overkill for a 5 GB Rails app's prod DB. Skip when the database is on a fully-managed cloud (RDS / Cloud SQL / Aurora / AlloyDB) where the provider already takes continuous backups with PITR — pgbackrest cannot run inside the managed instance and there is no archive command you control. Skip when logical, cross-version backups are the requirement (pre-major-version-upgrade dumps, schema-only exports, selective table extraction); pgbackrest does *physical* (file-level) backup only and a physical PG 16 backup cannot be restored into PG 17 — use `pg_dump` / `pg_dumpall` for that case. Skip if MySQL / MariaDB / MongoDB is the workload — pgbackrest is PostgreSQL-only by design; the equivalents are `xtrabackup` / `mariabackup` / `mongodump`.

## Vs already cataloged

- **Vs [`barman`](../barman/):** the other major open-source PG backup tool. Barman is Python with a daemon-style architecture and feature-set focused on a central backup server pulling backups via `streaming` or `rsync`; pgbackrest is C, command-driven, and pushes from the database host with a thinner architecture. Both are production-grade, both do PITR, both are recommended in the PostgreSQL ecosystem. pgbackrest tends to win on raw throughput and S3-native storage; barman tends to win where a single backup server centralizing many databases is the operating model.
- **Vs [`wal-g`](../wal-g/):** newer Go-based competitor with first-class S3 / GCS / Azure support and per-database extensibility (also supports MySQL, MongoDB, FoundationDB, Greenplum). wal-g is leaner and excellent if the workload is "PG to S3, single repo, single command"; pgbackrest is the more featureful choice when block-incremental, multi-repo, repo-side encryption, parallel restore-with-delta, or a long change-log of production scars are the requirements.
- **Vs [`pg_basebackup`](https://www.postgresql.org/docs/current/app-pgbasebackup.html):** orthogonal — `pg_basebackup` is the in-tree PostgreSQL utility for taking a single physical base backup (also used to bootstrap replicas). It does not manage WAL archiving, retention, incremental backup, or a backup catalog; pgbackrest does all of those on top of the same physical-backup primitive.
- **Vs [`restic`](../restic/) / [`kopia`](../kopia/) / [`rustic`](../rustic/) / [`borg`](../borg/):** general-purpose deduplicating backup tools. They can back up a stopped PG data directory, but they are not transaction-aware and cannot drive a PITR-correct WAL archive — using restic on `$PGDATA` of a running cluster produces an inconsistent snapshot. pgbackrest is the right answer when "PostgreSQL is the workload"; restic / kopia are right for "the rest of `/var` and `/etc`".
- **Vs [`pgcopydb`](https://github.com/dimitri/pgcopydb):** different goal. pgcopydb is for online logical migration (PG → PG with table re-shaping, parallel COPY, schema clone); pgbackrest is for backup / restore / PITR of the same major version. Use pgcopydb for major-version upgrades; use pgbackrest for ongoing operational backup.
- **Vs [`litestream`](../litestream/):** different database. litestream streams SQLite WAL to object storage; pgbackrest does the equivalent for PostgreSQL's WAL plus the base-backup tier.

## Caveats

- **Same major PG version on backup and restore.** Physical backups are tied to the on-disk format; a PG 17 backup restores to PG 17 (with point or any minor), not PG 16 or PG 18. Major-version upgrades go through `pg_dump` / `pg_upgrade`, not pgbackrest. Plan the upgrade-to-restore matrix accordingly.
- **`archive_command` correctness is critical.** PostgreSQL must be configured with `archive_mode = on` and `archive_command = 'pgbackrest --stanza=<name> archive-push %p'`; if archive-push fails, PostgreSQL retains WAL on the primary and the disk fills. Always pair with monitoring (`pgbackrest info --output=json` exposes the WAL min/max and the last backup age).
- **Repository capacity is your problem.** pgbackrest does not enforce a hard cap; the `--repo1-retention-full=N` / `--repo1-retention-diff=N` / `--repo1-retention-archive=N` knobs run during `expire` and decide what to delete. Misconfigured retention plus a chatty WAL workload can fill an S3 bucket fast.
- **Encryption key handling is on you.** `repo1-cipher-pass=<long-passphrase>` lives in the pgbackrest config; lose the passphrase and the backups are unrecoverable. Store the passphrase in a real secret manager (Vault, AWS Secrets Manager, sops-encrypted file) and back *that* up out-of-band.
- **The TLS server mode (replacing SSH for remote pgbackrest invocation, GA in 2.40+) needs PKI you manage.** It is more secure and faster than the SSH transport but the cert rotation story is `openssl` + your config-management tool, not built in.
- **Backup performance is bounded by `process-max` × repository write speed.** A misconfigured `process-max=2` against a 16-vCPU box leaves 14 cores idle; a `process-max=16` against a single S3 bucket with rate-limited multipart upload thrashes. Tune per environment and watch `pgbackrest info`'s reported throughput.
- **JSON output (`--output=json`) is the supported machine-readable surface.** Do not parse human-output text; it has changed across releases and there is no compatibility promise. The JSON schema is documented and stable.
- MIT ([LICENSE](https://github.com/pgbackrest/pgbackrest/blob/main/LICENSE)) — permissive; no copyleft on the database content or on the operator's config; safe in commercial deployments. Crunchy Data sponsors the project but the open-source surface and the binaries in the PGDG repositories are the canonical distribution.

## Example invocations

```bash
# Install
brew install pgbackrest
# or, on Debian / Ubuntu with PGDG:
apt install pgbackrest

# /etc/pgbackrest/pgbackrest.conf (one stanza, S3 repo, encrypted)
cat > /etc/pgbackrest/pgbackrest.conf <<'INI'
[global]
repo1-type=s3
repo1-path=/pg-backups/cluster-prod
repo1-s3-bucket=acme-pg-backup
repo1-s3-region=us-west-2
repo1-s3-endpoint=s3.us-west-2.amazonaws.com
repo1-s3-key-type=auto
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=<long-random-passphrase-from-vault>
repo1-retention-full=4
repo1-retention-diff=14
repo1-retention-archive=14
process-max=8
compress-type=zst
compress-level=3
log-level-console=info
log-level-file=detail
start-fast=y
archive-async=y
backup-standby=y

[main]
pg1-path=/var/lib/postgresql/18/main
pg1-port=5432
pg1-user=postgres
pg1-socket-path=/var/run/postgresql
INI

# postgresql.conf must have:
# archive_mode = on
# archive_command = 'pgbackrest --stanza=main archive-push %p'
# wal_level = replica   (or higher)

# Initialize the stanza (one-time)
pgbackrest --stanza=main stanza-create

# Take a full backup
pgbackrest --stanza=main --type=full backup

# Incremental (block-level) backup nightly
pgbackrest --stanza=main --type=incr backup

# Inspect the backup catalog
pgbackrest info
pgbackrest --output=json info | jq '.[0].backup[-1] | {label, type, timestamp, info}'

# Verify backups against the manifest (catch silent corruption)
pgbackrest --stanza=main verify

# PITR — restore to a specific timestamp on a recovery server
pg_ctl -D /var/lib/postgresql/18/main stop
pgbackrest --stanza=main --delta \
  --type=time --target='2026-04-30 12:34:56-07' \
  --target-action=promote restore
pg_ctl -D /var/lib/postgresql/18/main start

# Expire old backups per the retention policy
pgbackrest --stanza=main expire

# Build a fresh standby from the latest backup (delta restore on top of an existing $PGDATA)
pgbackrest --stanza=main --delta \
  --type=standby --recovery-option=primary_conninfo='host=primary user=replicator' restore
```
