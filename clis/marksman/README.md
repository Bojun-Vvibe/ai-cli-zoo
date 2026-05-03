# marksman

> **Markdown LSP server** — a single F# / .NET binary that speaks
> the Language Server Protocol over stdio for plain Markdown,
> CommonMark, and the wiki-link dialects used by Obsidian /
> Foam / Dendron / Zettlr / Logseq. Provides cross-file completion
> for `[[wiki-links]]` and `[label](path)`-style references,
> go-to-definition / find-references / rename across a workspace
> of `.md` files, document and workspace symbol search over
> headings and tags, fold and outline view, diagnostics for
> broken links and dangling references, and code actions for
> "create the missing note." Pinned to release **2026-02-08**
> ([LICENSE](https://github.com/artempyanykh/marksman/blob/main/LICENSE),
> MIT).

Source: <https://github.com/artempyanykh/marksman>

## TL;DR

`marksman` is what you reach for when your Markdown notes become
a *graph* — a personal knowledge base, a Zettelkasten, a project
spec folder, an `/docs` tree with hundreds of cross-referenced
files — and the editor stops being able to tell you "this link
points to a file that does not exist" or "this heading is
referenced from three other notes." It plugs into any LSP-aware
editor (Neovim, Helix, VS Code, Emacs, Zed, Sublime via LSP) and
turns the notes folder into something more like a typed codebase:
hover on a `[[note]]` link to preview, F12 to jump to the file,
Shift-F12 to find every backlink, F2 to rename a note and update
all references. Works on plain CommonMark out of the box and
recognises the Obsidian / Wikilinks superset when it sees one
(detected per-folder).

## Install

```bash
# Homebrew (macOS / Linux)
brew install marksman

# from a release binary
curl -L -o marksman https://github.com/artempyanykh/marksman/releases/download/2026-02-08/marksman-macos
chmod +x marksman
sudo install marksman /usr/local/bin/

# Linux x64
curl -L -o marksman https://github.com/artempyanykh/marksman/releases/download/2026-02-08/marksman-linux-x64
chmod +x marksman
sudo install marksman /usr/local/bin/

# scoop on Windows
scoop install marksman

# nix
nix-env -iA nixpkgs.marksman

# from source (requires .NET 9 SDK)
git clone https://github.com/artempyanykh/marksman && cd marksman
dotnet publish -c Release -o out Marksman/Marksman.fsproj
sudo install out/marksman /usr/local/bin/

# verify
marksman --version
```

`marksman` is one self-contained native binary (~22 MB on Linux,
~44 MB on macOS — bundles the .NET runtime). No
`dotnet-runtime` install required.

## License

MIT — see [LICENSE](https://github.com/artempyanykh/marksman/blob/main/LICENSE).
Permissive; no attribution required.

## One Concrete Example

```bash
# 1. point an editor at the binary; in Neovim with nvim-lspconfig:
#    require('lspconfig').marksman.setup{}
# in Helix (languages.toml):
#    [[language]]
#    name = "markdown"
#    language-servers = ["marksman"]
#    [language-server.marksman]
#    command = "marksman"
#    args = ["server"]

# 2. open a folder of notes; marksman auto-detects the workspace
mkdir notes && cd notes
cat > index.md <<'MD'
# Index
- [[meeting-2026-04-22]]
- [[ideas/agent-loop]]
MD
mkdir ideas
cat > ideas/agent-loop.md <<'MD'
# Agent loop
See also: [[../index]]
MD

# 3. open index.md in your editor — marksman will:
#    - underline [[meeting-2026-04-22]] as a broken link (file does not exist)
#    - resolve [[ideas/agent-loop]] and offer hover preview / go-to-def
#    - offer a code action "Create note 'meeting-2026-04-22'" on the broken link

# 4. one-shot CLI invocations (no editor) for scripting / CI
marksman --version
# Use it as an LSP from any editor; the binary itself is mostly an LSP server.
# A typical CI lint pass uses an external markdown linter (markdownlint-cli2)
# alongside marksman in-editor — they are complementary.
```

## Niche It Fills

**LSP-grade navigation for a Markdown knowledge graph, in any
editor.** Obsidian / Logseq / Dendron / Foam each ship excellent
graph navigation but lock you into their app or VS Code. Plain
editors (Neovim / Helix / Emacs / Zed) treat Markdown as text:
no completion for `[[`-links, no rename-across-files, no
backlink list, no dangling-link diagnostic. `marksman` is the
missing language server that gives any LSP-aware editor the
"notes-as-typed-codebase" experience without joining an
ecosystem. Bring your own editor, point it at `marksman`, get
graph navigation.

## Why use it

Three concrete capabilities that change the daily workflow:

1. **Cross-file completion + diagnostics for wiki-links.** Type
   `[[` and get a fuzzy-completion list of every note in the
   workspace; save a file with `[[broken-link]]` and get a
   diagnostic underline within milliseconds. Notes graphs only
   stay coherent if broken links surface immediately — without
   this, a folder of 500 notes silently rots.
2. **Workspace-wide rename.** F2 on a note's filename or heading
   updates every reference across the workspace atomically,
   the same way `tsserver` renames a TypeScript symbol. The
   single biggest reason notes graphs become unreadable over
   time is "I renamed a file and now half the links are
   broken"; LSP rename eliminates that failure mode.
3. **Find-references = backlinks for free.** Shift-F12 on a
   heading or note shows every `.md` file that links to it, in
   a quickfix list / location list / sidebar. This is the
   "backlinks panel" Roam / Obsidian made famous, available in
   any editor that speaks LSP.

For an LLM-augmented notes workflow where an agent
([`fabric`](../fabric/) / [`llm`](../llm/) /
[`files-to-prompt`](../files-to-prompt/) /
[`code2prompt`](../code2prompt/)) generates and edits Markdown,
having the editor surface broken-link diagnostics on the
agent's output catches "the model invented a `[[note]]` that
does not exist" the moment the file is saved.

## Vs Already Cataloged

- **Vs [`mdbook`](../mdbook/):** orthogonal — `mdbook` is a
  *static-site generator* that turns a folder of Markdown into
  an HTML book; `marksman` is an *editor server* that helps you
  navigate the source. They compose: edit notes with
  `marksman`-aware editor, ship them to readers via `mdbook`.
- **Vs [`marksman`-style markdown linters
  `markdownlint-cli2`](../markdownlint-cli2/):** orthogonal —
  `markdownlint-cli2` enforces *style* rules (heading levels,
  list indent, line length); `marksman` is about *graph
  semantics* (link resolution, refactor, navigation). Run both:
  `markdownlint-cli2` in CI as a style gate, `marksman` in the
  editor as a navigation aid.
- **Vs [`mdcat`](../mdcat/) / [`glow`](../glow/) /
  [`frogmouth`](../frogmouth/):** different surface — those are
  *renderers* for reading Markdown in a terminal; `marksman` is
  the LSP for *authoring* across many files. Use a renderer to
  read a single rendered note; use `marksman` while writing the
  graph.
- **Vs [`zk`](../zk/):** sister tool, partial overlap — `zk`
  is a CLI for a Zettelkasten that handles new-note templating,
  tag indexing, and `fzf`-style search; it is a *librarian*.
  `marksman` is the *editor server* that runs alongside while
  you are inside a file. Pair them: `zk new` to create notes,
  `marksman` to navigate them in the editor.
- **Vs [`mdtt`](../mdtt/) / [`harper`](../harper/):** orthogonal
  — `mdtt` edits Markdown tables, `harper` is a grammar /
  spelling LSP. Stack all three: `harper` for prose, `mdtt` for
  tables, `marksman` for graph navigation.
- **Vs Obsidian / Logseq (not cataloged):** they are full apps
  with sync, plugins, daily-notes UI, mobile clients;
  `marksman` is a 22 MB binary that bolts onto your existing
  editor. Pick Obsidian for the curated app experience; pick
  `marksman` to stay in Neovim / Helix / Emacs.

## Caveats

- **Wiki-link dialect detection is heuristic.** `marksman`
  detects "Obsidian-style folder" by looking for `.obsidian/` or
  `.marksman.toml`; if neither is present it parses CommonMark
  links only and may not resolve `[[wiki-links]]` correctly.
  Drop a `.marksman.toml` in the workspace root to force a
  dialect.
- **No formatter.** `marksman` does not reformat Markdown
  (alignment, table normalisation, bullet style). Pair with
  `dprint`, `prettier`, or `markdownlint-cli2 --fix` for
  formatting.
- **Indexing cost on huge workspaces.** First open of a 5000+
  note workspace can take a few seconds while the index
  builds; subsequent edits are incremental and fast. The index
  is in-memory, so very large workspaces consume RAM
  proportional to file count.
- **No live preview server.** `marksman` does diagnostics + nav,
  not HTML preview. Pair with editor-native preview
  ([`glow`](../glow/) in a split, `mdcat`, VS Code's built-in)
  or a generator like [`mdbook`](../mdbook/).
- **F#/.NET runtime is bundled but binary is large.** The
  ~22 MB Linux / ~44 MB macOS binary is bigger than typical
  Go or Rust LSPs because the .NET runtime is statically
  embedded. Performance at runtime is fine; on-disk size is
  the cost.
- **No multi-root workspace support yet.** `marksman` indexes
  one root at a time; opening two unrelated notes folders in
  the same editor session means two `marksman` processes.
  Reasonable on a laptop, awkward in some editor configs.
