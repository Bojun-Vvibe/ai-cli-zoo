# dnote

> **A simple command-line notebook that organises notes into
> named "books" with full-text search, optional encrypted
> sync to a self-hostable server, and a daily-digest workflow
> that emails you back what you wrote** — every operation is a
> verb against a local SQLite store (`dnote add js "closures
> capture by reference"`), so capture stays inside the
> terminal.
> Pinned to **cli-v0.16.0** (released 2025-11-08,
> [`gh api repos/dnote/dnote/releases/latest`](https://github.com/dnote/dnote/releases/latest),
> [LICENSE](https://github.com/dnote/dnote/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/dnote/dnote>

## TL;DR

Note-taking CLIs split roughly three ways: **append-only
journals** ([`jrnl`](../jrnl/)) optimise for one chronological
stream — great for diary-style entries, lossy for
"closures capture by reference" snippets you want to *find again
by topic* six months later. **Markdown-vault TUIs**
([`nb`](../nb/), [`zk`](../zk/)) lean into Zettelkasten-style
linked notes with files on disk — powerful, but the friction of
"open editor → write file → save → commit" is too high for
30-second capture. **Sync-first proprietary apps** (Notion,
Obsidian, Bear) win on UI but lose on `cat ~/.dnote.db | grep`.
`dnote` lands in the gap: notes are atomic short entries
indexed by *book* (a string label like `js`, `vim`, `kubernetes`),
storage is a single SQLite DB, and the verbs are tiny —
`dnote add <book> "<note>"` for capture, `dnote view <book>` to
list, `dnote find <query>` for full-text search, `dnote edit
<book> <id>` to revise. There's an optional self-hostable Go
server (`dnote-server`, same repo) that the CLI can sync to over
HTTPS with end-to-end-encrypted payloads, and a hosted SaaS
(getdnote.com) that runs the same server. The product hook is a
*spaced-repetition email digest*: the server emails you a few
of your own old notes daily, so the notebook reads itself back
to you instead of becoming a write-only graveyard.

## Install

```bash
# Homebrew (macOS / Linux)
brew install dnote/dnote/dnote

# Go install (any platform with Go 1.21+)
go install github.com/dnote/dnote/pkg/cli@cli-v0.16.0

# Pre-built binary from a release
curl -L \
  https://github.com/dnote/dnote/releases/download/cli-v0.16.0/dnote_0.16.0_linux_amd64.tar.gz \
  | tar xz && sudo mv dnote /usr/local/bin/

# verify
dnote version
```

## Representative examples

```bash
# 1. One-shot capture — book name, then the note body
dnote add js "closures capture variables by reference, not by value"

# 2. Open $EDITOR for a longer note
dnote add vim

# 3. List all books, then list notes inside one
dnote view              # shows books + counts
dnote view js           # shows the notes in the 'js' book

# 4. Read one note in full (id from `view`)
dnote view js 17

# 5. Full-text search across every book
dnote find "spaced repetition"

# 6. Edit / remove
dnote edit js 17
dnote remove js 17

# 7. Self-hosted sync (after `dnote login` against your server)
dnote sync                                  # pushes local → remote, pulls remote → local
DNOTE_SERVER=https://notes.example.com \
  dnote login                               # one-time auth

# 8. Inspect / back up the raw store — it's just SQLite
sqlite3 ~/.dnote/dnote.db '.tables'
cp ~/.dnote/dnote.db ~/backups/dnote-$(date +%F).db
```

## When to use vs. alternatives

- Pick **dnote** when capture should be *one verb* with no
  editor open ("`dnote add k8s 'kubectl rollout undo'`" while a
  pager is still on screen), the search story is more important
  than rich Markdown, and you want optional E2E-encrypted sync
  *with* a self-hosted server option (Apache-2.0, Go binary, no
  Electron).
- Pick [`jrnl`](../jrnl/) when the workflow is *chronological
  journaling* (one tagged stream over time, "yesterday I…")
  rather than topic-bucketed snippets — `jrnl` wins the diary
  metaphor; `dnote` wins the cheatsheet metaphor.
- Pick [`nb`](../nb/) for the Markdown-vault model — files on
  disk, git-backed, attachments, bookmarks, TUI browser — when
  the notebook *is* a long-lived knowledge base you want to
  grep with `rg` and version with git. Heavier, more powerful;
  not for "thirty-second capture in a tmux pane".
- Pick [`zk`](../zk/) when the discipline is Zettelkasten —
  linked atomic notes with explicit `[[wikilinks]]`, full-text
  + tag + link queries — and the editor experience matters more
  than the CLI ergonomics.
- Pick [`khoj`](../khoj/) when the *retrieval* is the point and
  you want an LLM-grounded search over the same notes corpus —
  orthogonal layer; you can keep capturing in `dnote` and point
  Khoj at the export.
- Pick a hosted SaaS (Notion, Obsidian Sync) when team-share /
  rich-text / mobile capture dominate the requirement set;
  `dnote`'s mobile story is "open the web UI of your
  self-hosted server", not a polished native app.
- Caveats: pre-1.0 (v0.x — pin the binary in CI rather than
  tracking `@latest`), the spaced-repetition digest only fires
  if you run the server (the standalone CLI is local-only), the
  `cli-vX.Y.Z` tag scheme is intentional (the repo also tags
  the server separately as `web-vX.Y.Z`) so don't pin the wrong
  prefix in your `Brewfile`, and the project has had long quiet
  stretches between releases — fine for a personal notebook,
  weigh it for organisation-wide deployment.
