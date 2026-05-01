# ddev

**Repo:** https://github.com/ddev/ddev
**Version:** v1.25.2
**License:** Apache-2.0 — [LICENSE](https://github.com/ddev/ddev/blob/main/LICENSE)
**Language:** Go

## What it does

`ddev` is a Docker-based local development environment manager
focused on PHP and Node.js web stacks (Drupal, WordPress, Laravel,
Magento, TYPO3, Craft, Symfony, Backdrop, Silverstripe, Statamic,
plus generic PHP / Node projects). One `ddev config` command in a
project root introspects the codebase, picks a project type, writes
`.ddev/config.yaml`, and `ddev start` then brings up a per-project
container set (web + db + optional Redis / Solr / Elasticsearch /
Mailpit / phpMyAdmin) on a routable `*.ddev.site` hostname with TLS
certificates auto-issued by the bundled mkcert root. Each project is
isolated; `ddev list` shows the full fleet, `ddev poweroff` stops
everything cleanly.

## Install

```bash
# macOS
brew install ddev/ddev/ddev

# Linux (Debian/Ubuntu)
curl -fsSL https://ddev.com/install.sh | bash

# verify
ddev --version    # ddev version v1.25.2
```

A working Docker provider (Docker Desktop, OrbStack, Colima, Lima,
Rancher Desktop, or rootless Docker engine) is the only hard
prerequisite.

## Real usage example

Spin up a fresh WordPress site:

```bash
mkdir my-wp && cd my-wp
ddev config --project-type=wordpress --docroot=. --create-docroot
ddev start
ddev wp core download
ddev wp config create --dbname=db --dbuser=db --dbpass=db --dbhost=db
ddev wp core install --url=https://my-wp.ddev.site \
  --title="Local" --admin_user=admin --admin_password=admin \
  --admin_email=admin@example.com
ddev launch
```

That gets you a fully-resolved `https://my-wp.ddev.site` (real cert,
no browser warnings) backed by an isolated MariaDB container, with
`wp` available as a `ddev wp ...` shim that runs inside the container.

Useful day-two operations:

```bash
ddev describe                     # ports, URLs, container state
ddev ssh                          # shell into the web container
ddev import-db --src=dump.sql.gz  # restore a snapshot
ddev export-db --file=snap.sql.gz # take a snapshot
ddev xdebug on                    # toggle Xdebug for the IDE
ddev share                        # ngrok-style public tunnel for review
ddev poweroff                     # stop all projects, free RAM
```

## Why it's interesting (orthogonal niche)

The catalog already covers Docker / dev-environment ground from
several angles (`devbox`, `daytona`, `lima`, `nerdctl`, `colima`-adjacent
tooling), but `ddev` fills a gap none of those touch: it is the only
**CMS / PHP-app-shaped local environment manager** in the zoo, and
the one that ships an opinionated, runnable stack (TLS, hosts file,
mailcatcher, db, search index) with a single `ddev start`. Where
`devbox` gives you a reproducible shell and `daytona` gives you a
remote workspace, `ddev` gives a CMS developer the exact stack the
production server runs, with one-command database snapshots and
per-project hostnames — the local dev-loop slot for the
WordPress / Drupal / Laravel / TYPO3 ecosystem that no other CLI
in the catalog targets directly.
