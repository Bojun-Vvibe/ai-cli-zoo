# epy

> **Terminal ebook reader for EPUB / EPUB3 / MOBI / AZW / FB2** —
> a single-file Python TUI that opens an ebook, restores the last
> reading position, paginates the content reflowed to the terminal
> width, supports hjkl / arrow navigation, full-text search,
> bookmarks, in-text annotations, dictionary lookup, light/dark
> themes, and a contents-tree TOC sidebar. Pinned to **v2022.12.11**
> (SPDX: `GPL-3.0`,
> [LICENSE](https://github.com/wustho/epy/blob/master/LICENSE)).

Source: <https://github.com/wustho/epy>

## TL;DR

`epy` is one ~2,500-line Python file (no compiled deps beyond
`pip install epy-reader`) that turns the terminal into an ebook
reader: open `epy book.epub`, get a paginated reflowed view sized
to the current terminal, navigate with vim keys (`j/k` line,
`PgDn/PgUp` page, `g/G` start/end, `[/]` chapter, `Tab` TOC,
`/` search, `b` bookmark, `?` help), close, reopen six months
later on a different machine, resume on the exact paragraph.
The killer property is **a real ebook reader on a no-X server
or a `tmux` pane** — the answer for reading technical PDFs
exported as EPUB on a remote box, or for reading on a Linux
laptop without GUI ebook software, or in a tmux pane next to
the editor while taking notes.

## Install

```bash
# pipx (recommended — isolated env)
pipx install epy-reader

# pip
pip install --user epy-reader

# Arch Linux (AUR)
paru -S epy-reader

# verify
epy --version    # epy v2022.12.11
```

## License

GPL-3.0 — see
[LICENSE](https://github.com/wustho/epy/blob/master/LICENSE).

## One Concrete Example

```bash
# 1. open an EPUB and start reading
epy ~/Books/the-rust-programming-language.epub

# 2. resume the last book read (no path needed)
epy

# 3. list reading history (every book opened, last position)
epy -r
#    1: SICP-Structure-and-Interpretation.epub  ch.4 / 35%
#    2: the-rust-programming-language.epub      ch.12 / 64%
#    3: thinking-in-systems.epub                ch.7 / 88%

# 4. open the Nth book from history
epy 2

# 5. dump-mode: extract reflowed plain text to stdout
epy -d ~/Books/book.epub > book.txt

# in-app keys (most useful subset):
#   j / k           line down / up
#   Space / b       page down / up
#   PgDn / PgUp     page down / up
#   g / G           start / end of book
#   [ / ]           previous / next chapter
#   Tab             toggle TOC sidebar
#   /pattern        full-text search forward
#   n / N           next / previous match
#   b               toggle bookmark on current page
#   B               jump to bookmark list
#   d               look up word under cursor in `dict` (configurable)
#   = / -           increase / decrease screen padding
#   c               toggle dark / light theme
#   o               open the source file with $READER (mupdf, etc.)
#   ?               help overlay
#   q               quit (saves position automatically)
```

Config lives in `~/.config/epy/configuration.json` — set the
external dictionary command (`"DictionaryClient": "dict -d wn"`),
the colour theme, default padding, and the screen-shrink hotkeys.

## Niche It Fills

**A first-class ebook reader for the terminal.** Three workflows
nothing else in this catalog covers:
- Reading a technical book on a no-X server reached over SSH
  (the only alternatives are Calibre's GUI viewer, which needs
  X / Wayland, or `pandoc`-converting to plain text and losing
  chapter structure).
- Reading on a Linux laptop without installing Calibre /
  Foliate / Bookworm — `pipx install epy-reader` is one
  dependency-free command, no Qt / GTK / WebEngine pull-in.
- Reading in a `tmux` / [`zellij`](../zellij/) pane next to the
  editor and shell — the right shape for working through a
  textbook while writing notes in the adjacent pane, where a
  separate GUI window would steal focus and break the
  keyboard-only flow.

The catalog has [`mdcat`](../mdcat/) (renders Markdown in the
terminal), [`glow`](../glow/) (interactive Markdown TUI),
[`tectonic`](../tectonic/) (LaTeX → PDF), [`pandoc`](../pandoc/)
(format conversion), and [`pdfgrep`](../pdfgrep/) (search PDFs)
— but no EPUB/MOBI reader.

## Why use it

1. **One Python file, no compiled deps.** `pipx install
   epy-reader` works on any host with Python 3 — no
   Qt / GTK / WebEngine / WebKit, no Electron, no headless
   Chrome. Installs in under 10 s on a fresh box.
2. **Position persistence.** Every quit writes the current
   chapter + offset + bookmark set to a JSON state file, so
   resume-on-a-different-machine via dotfiles sync works the
   way `vim`'s `viminfo` does for editor state.
3. **Reflow at the terminal width.** Unlike PDF readers,
   text reflows to the current `tput cols` width — readable
   on a 60-column tmux pane next to a 120-column editor pane.
4. **TOC + full-text search + bookmarks.** Three navigation
   primitives are enough for books — `Tab` to TOC, `/` to
   search, `b` to bookmark — without the 50-key cheatsheet a
   richer GUI imposes.
5. **Multi-format.** EPUB / EPUB3 / MOBI / AZW / FB2 / HTMLZ —
   the union covers ~all DRM-free ebook sources (Project
   Gutenberg, Standard Ebooks, technical-publisher direct
   downloads, fan-translated FB2 archives).
6. **Dump mode for pipelines.** `epy -d book.epub | grep -n
   pattern` for ad-hoc full-text grep across an ebook
   collection without unpacking the EPUB by hand.

## Vs Already Cataloged

- **Vs [`mdcat`](../mdcat/) / [`glow`](../glow/):** Those render
  Markdown. `epy` reads ebooks (EPUB / MOBI / FB2) — different
  source format, different navigation needs (chapters,
  bookmarks, TOC sidebar). They compose only via `pandoc`
  conversion in either direction.
- **Vs [`pandoc`](../pandoc/):** `pandoc -f epub -t plain` dumps
  a book to text — usable but loses chapter boundaries, TOC,
  navigation, search-with-context, position memory. `epy` is
  the *interactive* layer `pandoc` does not aspire to.
- **Vs [`pdfgrep`](../pdfgrep/) / [`pdfcpu`](../pdfcpu/):** Those
  operate on PDFs and are search/manipulation tools, not
  readers. `epy` is reader-shaped for EPUB/MOBI; for PDFs use
  `mupdf` / `zathura` (GUI) or `pdftotext` + `epy -d` style
  pipeline.
- **Vs Calibre `ebook-viewer`:** Calibre is the canonical Linux
  GUI ebook viewer. Pick Calibre on a desktop where you also
  manage a library; pick `epy` on a remote SSH session, in a
  tmux pane, or on a minimal install where Qt is not welcome.
- **Vs [`browsh`](../browsh/) opening a web ebook viewer:**
  browsh + an HTML reader works but pulls a headless Firefox
  for what `epy` does in 5 MB of Python.

## Caveats

- **Update cadence is slow.** v2022.12.11 is the latest tagged
  release as of writing — the project is largely feature-
  complete and maintained-as-needed, not actively iterating.
  For most ebooks this is fine; new EPUB3 features
  (interactive scripts, embedded video) are not in scope and
  never will be.
- **No DRM support.** Adobe ADEPT / Amazon KFX / etc. are
  intentionally out of scope. Strip DRM with Calibre's
  `DeDRM` plugin first if you legally own DRM-protected
  copies — `epy` will not open encrypted files.
- **GPL-3.0.** Linking against `epy`'s internals from
  proprietary software requires GPL compliance. Using the CLI
  via shell pipelines has no such constraint.
- **Dictionary lookup is external.** `d` shells out to a
  configurable dictionary command (`dict`, `sdcv`, `trans`).
  Install one separately and configure
  `"DictionaryClient"` in the config file before that key
  works.
- **Image rendering is text-mode by default.** Inline images in
  EPUBs render as `[IMG: alt-text]` placeholders — fine for
  text-heavy books, less good for image-heavy ones. The image-
  capable terminals (kitty, wezterm, foot) need an external
  viewer hookup, not built-in support.
