# nb

- **Repository:** https://github.com/xwmx/nb
- **Latest version:** 7.25.4
- **License:** AGPL-3.0 — verified at [`LICENSE`](https://github.com/xwmx/nb/blob/master/LICENSE) (raw: https://raw.githubusercontent.com/xwmx/nb/master/LICENSE)
- **Niche:** Plain-text notes / bookmarks / tasks CLI with git-versioned, encryption-capable, multi-notebook storage

## What it does

`nb` is a single Bash script (no runtime, no daemon, no database)
that turns any directory into a versioned, full-text-searchable,
optionally-encrypted notebook of plain-text notes, Markdown
documents, archived web pages, code snippets, todo items, and
file attachments. Every change is a git commit; every notebook can
be its own remote.

```
nb add "shower-thought.md"                          # opens $EDITOR, commits on save
nb add --type todo "ship the migration"             # task with toggleable state
nb bookmark https://example.com/article             # archives the page locally + tags
nb search "regex|literal" --all                     # ripgrep across notebooks
nb notebooks add work git@github.com:me/work.git    # additional notebook with own remote
nb use work && nb sync                              # context-switch + push/pull
nb encrypt --password "$PW"                         # AES-256 per-notebook
```

`nb` ships with an interactive TUI (`nb browse`), web server
(`nb browse --serve`), bookmark archiver (single-file HTML +
plaintext + structured metadata), tasks layer, and tag system —
but the storage is always plain files in plain directories under
git, so the entire corpus survives `nb` itself disappearing.

## Why interesting

The note / bookmark / personal-knowledge space has converged on two
extremes: heavyweight all-in-ones with proprietary stores (Notion,
Evernote, Roam, Obsidian-with-sync) and "just use Markdown in a
git repo" with no UX. `nb` sits exactly in the middle: every
artifact is a file you could `cat`, `grep`, and `git log` without
the tool installed, but the day-to-day surface (`nb add`,
`nb edit 12`, `nb search`, `nb bookmark <url>`, `nb sync`) is
ergonomic enough that you actually use it.

The notebook abstraction is the load-bearing idea. Different
notebooks can have different remotes, different encryption keys,
different sync cadences — so "personal scratch on a private repo",
"team runbooks on a shared repo", and "client-X notes encrypted at
rest, never pushed" are three `nb notebooks add` calls, not three
different tools.

Bookmarks deserve a separate mention. `nb bookmark <url>` does not
just store the URL — it fetches the page, archives a single-file
HTML snapshot, extracts plaintext for search, captures title /
description / tags, and commits the lot. Link rot stops mattering
because the *content* is in your git history, not just the URL.

## Pairs well with

- [`khoj`](../khoj/) — when you want the same plain-text corpus
  *also* searchable via local-LLM semantic queries on top of `nb`'s
  ripgrep-style literal search.
- [`fzf`](../fzf/) / [`ripgrep`](../ripgrep/) — `nb`'s files are
  just files; the same fuzzy-find / regex tools you use for code
  work over the notebook directory verbatim.
- [`git-bug`](../git-bug/) / [`jj`](../jj/) — orthogonal "data lives
  in git refs" tools that compose well with `nb`'s
  one-notebook-per-repo model.

## When to skip

- You need **real-time multi-user collaboration** on the same note
  (cursors, presence, conflict-free merge) — `nb` is git-merge
  semantics, which is fine for asynchronous use and miserable for
  two people typing in the same file.
- AGPL-3.0 is incompatible with how you would embed it — `nb` is
  CLI-shaped so the AGPL question is rarely live, but if you plan
  to wrap it as a hosted service the network-use clause matters.
- Pure WYSIWYG / rich-media note-taking is a hard requirement —
  `nb` is plain-text-and-Markdown first; images and PDFs attach
  cleanly, hand-drawn diagrams and embedded canvases do not.
