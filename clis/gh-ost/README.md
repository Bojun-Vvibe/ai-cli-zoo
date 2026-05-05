# gh-ost

> Snapshot date: 2026-05. Upstream: <https://github.com/github/gh-ost>

**GitHub's triggerless online schema-migration tool for MySQL.** `gh-ost`
("**G**itHub **O**nline **S**chema **T**ransmogrifier") rewrites a large
MySQL/MariaDB table without locking it, without using triggers on the
source table, and with a continuously throttleable, **interactively
controllable** migration process. Instead of installing
`AFTER INSERT/UPDATE/DELETE` triggers (the approach `pt-online-schema-change`
takes), `gh-ost` connects as a replica, reads the binary log stream, and
replays row events against a shadow ghost table — which is why a
multi-hour `ALTER TABLE` on a 500 GB hot table can run on the primary
during business hours without taking the app down.

## Repo + version + license

- Repo: <https://github.com/github/gh-ost>
- Latest release: **`v1.1.9`** (2026-05-01)
- HEAD on `master`: `4d38923`
- License: **MIT** —
  <https://github.com/github/gh-ost/blob/master/LICENSE>
- License path in repo: `LICENSE` (SPDX: `MIT`)
- Default branch: `master`
- Language: Go

## Install

```bash
# Homebrew
brew install gh-ost

# Static release binary (linux/macOS, amd64/arm64) from
# https://github.com/github/gh-ost/releases

# Build from source
go install github.com/github/gh-ost/go/cmd/gh-ost@latest
```

## Hello-world usage

```bash
# Migrate `mydb.users`: add a column, no app downtime, throttle if
# replica lag exceeds 1.5s. Runs against the primary; gh-ost discovers
# a replica to read binlogs from automatically.
gh-ost \
  --user="schema_migrator" --password="$MYSQL_PW" \
  --host=primary.db.internal --database="mydb" --table="users" \
  --alter="ADD COLUMN last_seen_at DATETIME NULL, ADD INDEX idx_last_seen (last_seen_at)" \
  --max-load='Threads_running=25' \
  --critical-load='Threads_running=1000' \
  --chunk-size=1000 \
  --max-lag-millis=1500 \
  --throttle-control-replicas="replica1.db.internal,replica2.db.internal" \
  --initially-drop-ghost-table \
  --postpone-cut-over-flag-file=/tmp/gh-ost.users.postpone \
  --execute
```

```bash
# Interactive control via a Unix socket (created automatically):
echo status     | nc -U /tmp/gh-ost.mydb.users.sock
echo throttle   | nc -U /tmp/gh-ost.mydb.users.sock   # pause copy
echo no-throttle| nc -U /tmp/gh-ost.mydb.users.sock   # resume
echo chunk-size=2000 | nc -U /tmp/gh-ost.mydb.users.sock
# When you're ready for the (sub-second) cut-over:
rm /tmp/gh-ost.users.postpone
```

Drop `--execute` for a no-op dry run that still validates connectivity,
permissions, foreign-key safety, and schema compatibility.

## Niche

The "**online MySQL ALTER TABLE without triggers**" slot. The field is
small and the trade-offs are sharp:

- [`pt-online-schema-change`](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)
  (Percona Toolkit) — the older incumbent. Uses triggers on the source
  table, which couples migration overhead to write traffic and can
  surprise you under high concurrency.
- [`Spirit`](https://github.com/cashapp/spirit) (Cash App) — newer,
  also triggerless, parallelises better on very large tables but is
  less battle-tested in the wild than `gh-ost`.
- Vitess `OnlineDDL` / Aurora's native online DDL — only available if
  you're already on those platforms.
- Native MySQL 8 `ALGORITHM=INPLACE`/`INSTANT` — fine for the operations
  it covers, but anything touching primary keys, character sets, or
  large data rewrites still needs a tool like `gh-ost`.

`gh-ost` is the option you reach for when you want **production-proven
on a primary that's also serving live traffic**, with the ability to
*pause and resume the migration*, swap the cut-over to a maintenance
window, and abort cleanly at any point.

## Why it matters

- **Triggerless = predictable load profile.** Migration cost is paid by
  `gh-ost`'s own connection (binlog reader + chunked copy), not by every
  app-side `INSERT` paying for trigger evaluation. You can throttle
  *gh-ost* without touching the application.
- **Interactive cut-over.** The copy phase can run for hours; the actual
  table swap is sub-second and gated on a flag file or socket command.
  This is the feature that lets ops-on-call schedule the cut-over for
  3am after the heavy lifting already finished during the day.
- **First-class throttling signals.** `--max-load`,
  `--critical-load`, `--throttle-control-replicas`, and
  `--throttle-query` mean the migration *yields* to replication lag,
  thread pile-ups, or any custom SQL predicate you give it (e.g.
  "throttle if checkout queue depth > 500").
- **Used in anger by GitHub itself.** `gh-ost` was open-sourced *out
  of* GitHub's production schema-migration platform, and the project
  is still actively maintained — `v1.1.9` shipped this week.
- **MIT-licensed, single Go binary.** No agent to install on the DB
  host, no Perl runtime, no language stack to manage. Drop the binary
  on a bastion that can talk to MySQL on 3306 and you're done.
