# silverbullet

> **Markdown-native personal knowledge platform with a real
> Lua scripting surface** — a single self-hosted server binary
> (`silverbullet`) plus a separate headless-Chrome-driven CLI
> client (`silverbullet-cli`) that lets you evaluate Lua
> expressions, run scripts, and pull live console output
> against a running space, all over Markdown files on disk —
> pinned to **v2.6.1** (release
> [`2.6.1`](https://github.com/silverbulletmd/silverbullet/releases/tag/2.6.1)
> against `main`,
> [LICENSE.md](https://github.com/silverbulletmd/silverbullet/blob/v2.6.1/LICENSE.md),
> MIT).

Source: <https://github.com/silverbulletmd/silverbullet>

## TL;DR

The personal-knowledge-management (PKM) space splits roughly
into:

1. **Closed-source desktop apps with a sync service**: Notion,
   Obsidian, Bear, Craft. Either proprietary file format or
   proprietary sync, often both.
2. **Local-first Markdown editors with no programmability**:
   Typora, MarkText, iA Writer. Great editor, no automation —
   you can't query "all my pages tagged `#book` published this
   year and aggregate the ratings."
3. **Programmable PKM with a heavy stack**: Logseq (Electron
   + ClojureScript), Trilium (Electron + Node + SQLite),
   Anytype (Go + custom protocol). Powerful, but a lot of
   moving parts to self-host and keep updated.

`silverbullet` is the right shape when your real ask is
"Markdown files on disk, edited in a fast browser-native
editor, with a *real* embedded scripting language so I can
write queries, custom commands, and small automations against
my own notes." The space is just a directory of `.md` files —
you can `git`-track them, `rsync` them, edit them in `vim` on
the server, and `silverbullet` will pick the changes up. The
scripting surface is **Space Lua** — a real Lua 5.4 interpreter
embedded in the page-render path, with a query language
(LIQ — Lua Integrated Query) on top, so you can write
`from index.tag "book" select title, rating order by rating
desc` directly inside a page and the rendered output updates
live as your notes change.

## Install

```bash
# Pre-built single binary (Linux / macOS / Windows / FreeBSD,
# both x86_64 and aarch64; ARMv7 for Linux)
curl -LO https://github.com/silverbulletmd/silverbullet/releases/download/2.6.1/silverbullet-server-linux-x86_64.zip
unzip silverbullet-server-linux-x86_64.zip
./silverbullet --help

# Or Docker
docker pull ghcr.io/silverbulletmd/silverbullet:2.6.1

# The new (experimental) Runtime CLI client is a separate zip:
curl -LO https://github.com/silverbulletmd/silverbullet/releases/download/2.6.1/silverbullet-cli-linux-x86_64.zip
unzip silverbullet-cli-linux-x86_64.zip
./silverbullet-cli --help

# Verify
./silverbullet --version       # → 2.6.1
```

## Day one

```bash
# 1. Pick a directory for your space (just a folder of .md files)
mkdir -p ~/notes/my-space

# 2. Boot the server, bind locally, set a single-user password
SB_USER=me:hunter2 \
  ./silverbullet --port 3000 --hostname 127.0.0.1 ~/notes/my-space

# 3. Open http://127.0.0.1:3000, log in, start typing.
#    Every page you create lands as a real .md file under
#    ~/notes/my-space/, editable from any other tool.

# 4. Drive the running space from a script using the new CLI
SB_URL=http://127.0.0.1:3000 SB_USER=me:hunter2 \
  ./silverbullet-cli eval 'return #editor.getText()'

SB_URL=http://127.0.0.1:3000 SB_USER=me:hunter2 \
  ./silverbullet-cli run ~/scripts/nightly-cleanup.lua

# 5. Inside any page, embed a live query block:
#    ```query
#    from index.tag "book"
#    select title, rating
#    order by rating desc
#    ```
#    re-evaluates on every page render against your current notes.
```

The on-disk format is just CommonMark Markdown plus a small set
of frontmatter keys (`tags:`, `aliases:`, `pageDecoration:`).
Wikilinks are `[[Page Name]]`, tags are `#tag`, queries live in
fenced code blocks. There is no proprietary index file — the
index is rebuilt from your Markdown on startup and kept in a
small SQLite-or-equivalent KV store next to the space.

## Why pin to v2.6.1

- **Brand-new (experimental) Runtime API + `silverbullet-cli`
  client** — the first version where you can drive a running
  space from outside the browser: evaluate Lua, run scripts,
  read console logs over an HTTP API, all backed by a headless
  Chrome instance running the full client over CDP. This is
  the version where automation against your space stops
  requiring a screenscraper.
- Codebase migration from Deno to Node.js with chunked ESBuild
  bundles and JIT loading of large modules (vim, syntax modes)
  — meaningfully smaller cold starts.
- Outline operations (`Outline: Move Up/Down/Indent`) reworked
  to handle numbered lists, headers (moves entire sections),
  and paragraphs.
- Markdown footnote support (both `[^1]` reference-style and
  inline `^[…]`), with hover previews and broken-reference
  linting.
- 13 new aggregate functions in LIQ (`product`, `string_agg`,
  `yaml_agg`, `json_agg`, `bit_and/or/xor`, `bool_and/or`,
  `stddev_pop/samp`, `var_pop/samp`) plus implicit single-group
  for aggregates without `group by` and `offset` clause
  support.
- Lua interpreter hot-path optimizations and `LuaTable`
  internals tuning — observable speedup on pages that fan out
  many query blocks.

## When NOT to reach for it

- **You want offline-first, no-server, no-account.** The
  default deployment is a long-running HTTP server you point a
  browser at. There is a single-user mode (`SB_USER=…`) and
  you can run it on `127.0.0.1`, but there is no "double-click
  a desktop app" experience — for that, look at Obsidian or
  Logseq Desktop.
- **Your notes are mostly non-Markdown** (rich tables, embedded
  spreadsheets, hand-drawn diagrams that aren't excalidraw).
  The on-disk format is Markdown; anything that doesn't
  serialize to Markdown either renders weakly or doesn't
  round-trip.
- **You need real-time multi-user collaboration** (à la Notion
  / Google Docs). Multiple users can have accounts and edit
  different pages, but there is no operational-transform layer
  — concurrent edits to the same page will last-write-wins.
- **You're allergic to JavaScript-heavy frontends.** The
  editor is CodeMirror 6 in the browser; the server is small,
  but the client is a real SPA. If your bar is "must work in
  `lynx`," this is not the tool.
- **You need certified at-rest encryption / RBAC / audit
  logs.** The auth model is "one user, or reverse-proxy auth";
  the storage model is "plain Markdown files in a directory."
  If you need multi-tenant isolation, wrap it.

If those exclusions don't apply, `silverbullet` is the rare PKM
that gives you Markdown-on-disk *and* a real scripting language
*and* a CLI surface — without a SaaS account or an Electron
binary on your dock.
