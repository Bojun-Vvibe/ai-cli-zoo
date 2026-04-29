# zk

> **A plain-text, Zettelkasten-style notebook CLI that turns a directory
> of Markdown files into a queryable, link-aware knowledge base — with
> full-text search, tag/link graph traversal, fzf-powered interactive
> picking, and a built-in LSP so editors get completion on `[[wiki
> links]]` and tags.** Pinned to **v0.15.3**, GPL-3.0
> ([LICENSE](https://github.com/zk-org/zk/blob/main/LICENSE)).

- **Repo:** https://github.com/zk-org/zk
- **Latest version:** v0.15.3
- **License:** GPL-3.0 (`LICENSE` at repo root, SPDX `GPL-3.0`)
- **Category:** `notes` / `knowledge-base` / `language-server`
- **Language:** Go
- **Install:** `brew install zk` · `pacman -S zk` · `nix-env -iA
  nixpkgs.zk` · prebuilt binaries on the GitHub release page · binary
  name is `zk`

## What it does

`zk` treats a directory of Markdown files as a database. `zk init` in
an empty directory writes a `.zk/` folder with a SQLite index and a
`config.toml`; from then on every `zk new` creates a note with a
configurable filename template (timestamp, slug, ULID, custom Handlebars),
front-matter, and an optional opening prompt, while `zk` (no args) drops
you into an fzf picker over every note in the notebook with a live
preview pane.

The interesting surface is the query layer:

- **`zk list`** — filter notes by tag (`--tag inbox,!archived`),
  by link relationship (`--linked-by note.md`, `--link-to note.md`,
  `--orphan`, `--mentioned-by`), by full-text search (`--match
  "embedding model"`, ranked by SQLite FTS5), by date
  (`--created-after "last week"`), by path glob, or any combination.
  `--interactive` pipes the result into fzf.
- **`zk graph --format json`** — dumps the full link graph for
  downstream visualisation or LLM context-building.
- **`zk edit`** — same filters, opens matches in `$EDITOR`.
- **`zk lsp`** — a Language Server Protocol implementation. Wire it
  into Neovim/Helix/VS Code and you get completion on `[[wiki links]]`
  (against existing note titles), completion on `#tags`, hover preview
  of linked notes, "go to definition" on a wiki link, and diagnostics
  for broken links.
- **`zk new --interactive`** — runs a configured *group* template
  (a `[group]` block in `config.toml` with its own dir, filename
  template, body template, and extra metadata) so "daily note",
  "meeting note", "literature note", "project note" each have their
  own shape without separate scripts.

The whole thing is one Go binary with no daemon, no server, no
sync — files are plain Markdown on disk, the SQLite index is
disposable and rebuilds from `zk index`. Sync via git, Syncthing,
iCloud, or whatever the rest of your dotfiles use.

## Why included

The PKM (personal knowledge management) market has bifurcated into
heavy GUI apps with proprietary formats (Notion, Obsidian's `.obsidian`
directory, Logseq's graph DB) and bare-Markdown directories with no
tooling. `zk` is the practical middle: your notes stay plain Markdown
files in a git repo (so `grep`, `ripgrep`, `fzf`, `bat`, every other
tool in this catalog still works), but you get a real link graph, a
real query language, and a real LSP. The Zettelkasten primitives
(unique IDs, dense bidirectional linking, atomic notes) are
encouraged but not enforced — you can also use it as a flat journal
or a meeting-notes archive.

In an AI-native workflow this is the substrate piece: notes in a
predictable location with a queryable graph are exactly what
[`khoj`](../khoj/), [`fabric`](../fabric/),
[`files-to-prompt`](../files-to-prompt/),
[`repomix`](../repomix/), and [`code2prompt`](../code2prompt/)
need to slurp in as LLM context. `zk graph --format json` plus
`zk list --linked-by current-note --format path` give an agent the
"what does this note transitively touch" view without parsing
Markdown by hand. Pairs well with [`fzf`](../fzf/) (which `zk` already
uses internally), [`bat`](../bat/) (preview pane), and
[`tealdeer`](../tealdeer/) for the terminal-native muscle memory.

## Tradeoffs

- GPL-3.0 is the strictest license in this catalog. Linking `zk` as
  a library into a closed-source product is a non-starter; using the
  CLI as an external process from any other software is fine.
- No built-in sync. You are responsible for git-committing,
  Syncthing-ing, or otherwise replicating the directory. This is a
  feature for terminal-native users and a hard cliff for anyone
  who wants "open the iOS app and the note appears".
- The LSP requires editor configuration — works out of the box in
  Neovim with `nvim-lspconfig`, needs a manual server entry in
  Helix's `languages.toml`, ships as a VS Code extension. Not yet
  packaged for every editor.
- Filename templates are powerful but the Handlebars dialect has its
  own quirks; expect ten minutes of trial-and-error to land on the
  scheme you want for the next decade.
- Single-user by design. Multi-author notebooks work via git but
  there is no merge-conflict-aware UI.
