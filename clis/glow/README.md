# glow

> **Render Markdown on the CLI, with pizzazz** — a Charm Bracelet
> Go binary that takes a `.md` file (or stdin, or a URL, or a
> GitHub repo path) and renders it as styled, paged terminal
> prose: real headings, soft-wrapped paragraphs, syntax-
> highlighted code fences, table layout, and a TUI library browser
> for files under the current tree. Pinned to **v2.1.2** (commit
> `53788271b387c23b20b0db69fea2fd552593623b`,
> [LICENSE](https://github.com/charmbracelet/glow/blob/master/LICENSE),
> MIT).

Source: <https://github.com/charmbracelet/glow>

## TL;DR

`cat README.md` shows you the source bytes; `bat README.md` shows
you the source bytes with syntax colour. `glow README.md` shows
you what the rendered Markdown actually *says* — headings as
larger / bolder text, links unfurled (`[text](url)` becomes
underlined `text` with the URL on hover / inspect), code blocks
syntax-highlighted via Chroma, lists with proper bullets and
indent, tables aligned, blockquotes rendered with a left bar,
images shown as alt-text placeholders, and the whole thing soft-
wrapped to your terminal width and piped through a built-in
pager. Three operating modes: one-shot rendering (`glow file.md`),
a TUI library mode (`glow` with no args opens an `fd`-backed
browser of every `.md` under the cwd, with a stash for marking
favourites), and remote fetch (`glow github.com/owner/repo` or
`glow https://example.com/foo.md` pulls and renders without you
cloning anything). Themes: dark / light / dracula / pink / no-
colour (`auto` detects from the terminal). Cross-platform
(Linux, macOS, FreeBSD, Windows).

## Install

```bash
# Homebrew (macOS / Linux)
brew install glow

# Go
go install github.com/charmbracelet/glow@latest

# Linux package managers
# Arch: pacman -S glow
# Debian / Ubuntu (Charm apt repo):
#   echo 'deb [trusted=yes] https://repo.charm.sh/apt/ /' \
#     | sudo tee /etc/apt/sources.list.d/charm.list
#   sudo apt update && sudo apt install glow
# Fedora (Charm yum repo): see https://charm.sh/install
# Nix: nix-env -iA nixpkgs.glow

# Windows
# scoop install glow
# choco install glow
# winget install charmbracelet.glow

# release tarball (any OS / arch)
curl -Lo glow.tar.gz "https://github.com/charmbracelet/glow/releases/download/v2.1.2/glow_Darwin_arm64.tar.gz"
tar xf glow.tar.gz && sudo install glow /usr/local/bin/

# verify
glow --version    # glow version v2.1.2
```

First run drops a small config at `~/.config/glow/glow.yml` listing
the active style and pager binary; nothing else required. The TUI
library mode (`glow` with no args) walks the current directory
once and caches the file list in memory; on huge trees pass
`--all` or run from a narrower root.

## License

MIT — see
[LICENSE](https://github.com/charmbracelet/glow/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. render a single file (auto-paged when long)
glow README.md

# 2. read the README of a remote repo without cloning
glow github.com/charmbracelet/glow

# 3. render a markdown URL straight from the web
glow https://raw.githubusercontent.com/openai/openai-python/main/README.md

# 4. open the TUI library — browse every .md under cwd
glow                        # arrow keys + enter to open, s to stash, q to quit

# 5. force a specific style (dark / light / dracula / pink / notty / auto)
glow -s dracula CHANGELOG.md

# 6. render to a fixed width (good for screenshots / consistency)
glow -w 80 ARCHITECTURE.md | tee out.txt

# 7. read markdown from stdin (compose with other tools)
curl -s https://api.github.com/repos/charmbracelet/glow/releases/latest \
  | jq -r '.body' | glow -

# 8. pipe an LLM CLI's markdown output through glow for nicer reading
llm "Write a tutorial on Rust traits in markdown" | glow -

# 9. set glow as the default markdown viewer in your shell
alias mdr='glow'
```

## Niche It Fills

**The "*read* this Markdown, do not look at the source" viewer.**
README files, CHANGELOGs, design docs, RFCs, LLM-generated
explanations are written as Markdown but consumed as prose; on a
TTY without a browser the choice is "scroll through hashes and
backticks" or "switch to a graphical viewer". `glow` is the third
option — render in place, in the terminal, with the same key-
bindings as `less` and a width that matches your window. It pairs
with [`bat`](../bat/) the way `less` pairs with `cat`: `bat` is
the right answer when you want to see the *source* (about to edit,
or debugging the markdown itself), `glow` is the right answer when
you want to see the *rendered* output (about to read, or showing
someone a doc).

## Why use it

Three things `glow` does that other terminal markdown tools do
not, that pay back the install cost:

1. **Real Markdown rendering, not just colour.** Headings get
   different weights and underlines; lists get aligned bullets
   and proper hanging indent; tables are width-balanced and
   aligned; blockquotes get a left-bar; emphasis (`*…*` /
   `**…**`) becomes italic / bold; horizontal rules become full-
   width lines; images become alt-text placeholders. Powered by
   `glamour`, the same renderer used inside `gh`, `crush`, and
   most Charm TUIs — the rendering is consistent across the
   whole Bracelet ecosystem.
2. **Remote fetch is first-class.** `glow github.com/owner/repo`
   pulls the README of a repo without `git clone`, `glow
   <https-url-to-md>` works for any markdown file on the web,
   and the TUI library has a "stashed" view for files you have
   marked. The "I want to skim three READMEs before deciding
   which library to use" workflow is one command per repo, no
   browser tab, no cloning.
3. **Code fences get language-aware syntax highlighting.**
   ` ```rust ` / ` ```ts ` / ` ```bash ` blocks render with
   Chroma colour, the same engine [`bat`](../bat/) uses; comments
   stay grey, keywords stay coloured, strings stay strung — so a
   tutorial-shaped Markdown file (a README walking through code
   examples) reads the way it reads on GitHub, not the way it
   reads in `less`.

For an LLM-CLI workflow the killer pipe is `llm "Explain X in
markdown" | glow -` — model writes markdown, `glow` renders it,
you read prose with code blocks rendered as code, no copy-paste
into a browser, no markdown-to-HTML round trip.

## Vs Already Cataloged

- **Vs [`bat`](../bat/):** orthogonal — `bat README.md` shows
  syntax-highlighted *source* (you see the `#`, `**`, `[](…)`);
  `glow README.md` shows *rendered* output (the `#` becomes a
  heading, the `**` becomes bold, the link becomes underlined
  prose with the URL hidden). Use `bat` when you are about to
  edit the markdown; use `glow` when you are about to read it.
  The two are designed to coexist.
- **Vs [`gum`](../gum/):** orthogonal — both Charm Bracelet, but
  `gum` is a *toolkit of TUI primitives* (`gum confirm`, `gum
  choose`, `gum spin`, `gum input`) for building shell scripts
  with nice prompts; `glow` is a *finished application* for one
  job (rendering Markdown). Same design language, different
  layers of the stack.
- **Vs `mdcat` / `mdless` / `mdr` (not cataloged):** all three
  do roughly the same job. `mdcat` is one-shot, no TUI library,
  faster cold start, optional inline image support via `kitty`
  / `wezterm` graphics protocols; `mdless` is older, paged-only,
  weaker code-fence colouring; `mdr` is minimal Go, no remote
  fetch. `glow` is the most batteries-included of the set:
  remote fetch, TUI library, stash, themes, and the consistent
  Charm rendering engine. Pick `mdcat` if you want inline images
  and a smaller binary; pick `glow` for the library / remote-
  fetch UX.
- **Vs `pandoc README.md -t plain | less`:** `pandoc` strips
  formatting to plain text — you lose colour, you lose code
  highlight, you lose alignment. `glow` keeps all three.

## Caveats

- **No inline images.** `glow` renders images as alt-text
  placeholders, not as actual pixels in the terminal. Even on
  graphical terminals (`kitty`, `wezterm`, `iTerm2`) that
  support image protocols, `glow` does not use them; for that
  reach for `mdcat` with the `--image-protocol` flag.
- **TUI library walks the whole tree.** First launch in a large
  directory (e.g. a monorepo root) walks every directory looking
  for `.md` files; on a 100k-file tree this is several seconds.
  Run from a narrower root or pass a glob to limit scope.
- **Remote fetch is unauthenticated by default.** `glow
  github.com/private-org/repo` fails on private repos with a
  generic "404"; for authenticated fetch you need the file
  contents already on disk (clone first, then `glow`) or a
  pre-signed URL. There is no `--token` flag.
- **Width auto-detect can be wrong inside multiplexers.**
  Inside `tmux` / `zellij` panes that have been resized, `glow`
  occasionally picks the parent terminal width instead of the
  pane width and wraps too wide; pass `-w 100` (or whatever)
  explicitly when this happens.
- **Style is global, not per-file.** The active theme applies to
  every render in the session; you can not mark "this file is
  light, that file is dark". Workaround: shell alias per theme
  (`alias glow-light='glow -s light'`).
- **GitHub-flavoured Markdown extensions are partial.** Task
  lists (`- [x] foo`) render correctly; GitHub footnotes,
  mermaid blocks, and `<details>` collapsibles do not — they
  appear as fenced code or raw HTML. For full GFM parity you
  still want a browser.
