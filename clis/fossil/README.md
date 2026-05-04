# fossil

> **Self-contained DVCS + bug tracker + wiki + forum + tech-notes
> in one ~6 MB binary** — created by D. Richard Hipp (of SQLite)
> and used to host SQLite itself. Pinned to **v2.25** (released
> 2024-12-12, [COPYRIGHT](https://fossil-scm.org/home/file?name=COPYRIGHT-BSD2.txt),
> 2-clause BSD).

Source: <https://fossil-scm.org/home/doc/trunk/www/index.wiki>
(canonical), mirror at <https://github.com/drhsqlite/fossil-mirror>.

## TL;DR

`fossil` is a single statically-linked C binary that stores an
entire project — source history, tickets, wiki pages, a forum, a
chat room, technical notes, and unversioned attachments — in one
SQLite database file (`*.fossil`). Cloning is one HTTP round
trip; serving is `fossil serve` (built-in webserver, no nginx);
backing up is `cp project.fossil backup.fossil`. Everything ships
in-band: open the local UI with `fossil ui` and you immediately
get tickets, wiki, timeline, branch graphs, and diffs without
installing GitLab, Gitea, Trac, or any third-party tracker. The
storage format and HTTP sync protocol are stable and documented;
the same binary that started the project in 2007 still reads
today's repos.

## Why pick it over alternatives

Pick `fossil` when the *project* is small-to-medium, the *team*
is small-to-medium, and you want to stop running four services
(git server + issue tracker + wiki + CI dashboard) for a
two-person codebase. Compared to `git`: fossil is auto-sync by
default (push on commit), has built-in tickets/wiki/forum, and
the entire repo is one file you can email; git is faster on
million-commit monorepos and has the entire industry's tooling
around it. Compared to `sapling` / `jj`: those are git-protocol
client experiments — fossil is a different *server* model
entirely. Compared to `gitea` / `forgejo`: those are
git-hosting servers (multi-tenant, web-first); fossil is
single-project and works equally well as a local-only repo with
no server at all. Skip fossil if you need GitHub-style PR review
ergonomics, third-party CI integrations, or a community larger
than ~10 maintainers — the ecosystem outside the SQLite/Tcl
world is thin.

## Install

```bash
# macOS
brew install fossil

# Debian / Ubuntu
sudo apt install fossil

# verify
fossil version    # 2.25
```

Quick start:

```bash
# create a new repo
fossil init myproject.fossil
mkdir myproject && cd myproject
fossil open ../myproject.fossil

# add files, commit
fossil add .
fossil commit -m "initial import"

# launch the integrated web UI (tickets, wiki, timeline, diffs)
fossil ui
# opens http://localhost:8080 in your browser

# clone over HTTP
fossil clone https://example.com/repo.fossil ./local.fossil
```
