# rich-cli

> **Pretty-print anything from the shell using the `rich` library.**
> Markdown, JSON, syntax-highlighted source, CSV tables, inline
> images — all rendered to a truecolor terminal by a single command.
> Pinned to **v1.8.1**
> ([LICENSE](https://github.com/Textualize/rich-cli/blob/main/LICENSE),
> MIT).

Source: <https://github.com/Textualize/rich-cli>

## TL;DR

`rich-cli` is the command-line skin over the well-known Python
`rich` library. You hand it a path or stdin and a `--<format>`
flag (or it auto-detects from the file extension) and it emits a
themed, terminal-width-aware rendering: Markdown with proper
heading levels and code-fence syntax highlighting, JSON / YAML /
TOML pretty-printed and colorised, source code with Pygments-
quality highlighting in 100+ languages, CSV / TSV as a real
boxed table with column-type inference, and PNG / JPEG via
unicode-block downsampling. Everything is one process, one
config, one `pip install` — no chain of `bat | jq | mdcat | csvlook`.

## Install

```bash
# pipx (recommended — isolates the rich/Pygments/click deps)
pipx install rich-cli

# pip into the active env
pip install rich-cli

# Homebrew
brew install rich-cli

# verify
rich --version          # 1.8.1
```

## License

MIT — see
[LICENSE](https://github.com/Textualize/rich-cli/blob/main/LICENSE).
Permissive, embed-friendly. Safe to ship inside an internal tool
or a Docker image; the only attribution requirement is preserving
the notice. The bundled `rich` and Pygments dependencies are
also MIT / BSD-style.

## One Concrete Example

```bash
# 1. Render a Markdown README with proper headings + fenced code
rich README.md
# headings get ANSI styling, code blocks get language-aware
# syntax highlighting, tables get unicode borders, links get
# OSC-8 hyperlink escapes (clickable in modern terminals).

# 2. Pretty-print JSON from a pipe with collapsible structure
curl -s https://api.github.com/repos/Textualize/rich-cli | rich - --json
# arrays / objects indented, keys / strings / numbers / booleans
# colorised distinctly, trailing-comma-tolerant.

# 3. Syntax-highlight a source file as if it were in your editor
rich src/foo.py --theme monokai --line-numbers --guides
# Pygments lexer auto-picked from extension; --guides draws the
# indent guides; --theme is any Pygments style name.

# 4. Render a CSV as a real table, not as raw text
rich data.csv --csv
# inferred column types right-align numbers, header row gets
# bold styling, table is truncated to terminal width with
# ellipsis instead of horizontal scroll.

# 5. Show a PNG inline in the terminal (unicode half-block)
rich logo.png --width 60
# downsampled to 60 cols using the upper / lower half-block
# trick, so it works in any truecolor terminal without sixel /
# kitty graphics protocol support.

# 6. Force-render to ANSI for piping into less / a log
rich --force-terminal --width 100 README.md | less -R
# bypass isatty() detection so colors survive the pipe; -R in
# less passes the ANSI through.
```

## Niche It Fills

**The "I have a thing in the terminal and I want it to look the
way it does in my editor" gap.** Most shells give you `cat`,
which dumps raw bytes, and `less`, which adds paging but no
formatting. The decent renderers each cover *one* format —
[`bat`](../bat/) for source / Markdown, `jq` for JSON, `csvlook`
for tables, `mdcat` for Markdown, `glow` for Markdown, `chafa`
for images. `rich-cli` is the single binary that covers *all of
them* with one consistent theme system, one width-handling
policy, and one OSC-8 hyperlink convention. That matters when
you're previewing arbitrary files that an LLM agent or a `find`
pipeline just emitted: you don't know up front whether the next
file is `.md`, `.json`, `.py`, or `.csv`, and you don't want six
different format-detection wrappers.

## Why use it

Three things `rich-cli` does that the obvious alternatives don't:

1. **One binary, one theme.** Set `--theme monokai` once and
   Markdown code-fences, source files, JSON, and tables all
   harmonise. Switching between `bat` + `jq` + `csvlook` means
   each tool ships its own theme system you have to configure
   independently, and the colors will *not* match.
2. **Auto-detect by extension *and* by stdin shape.** `rich
   foo.json` infers JSON; `cat foo.json | rich - --json` does
   the same on a pipe; `rich foo.unknown --syntax python` lets
   you override. Most single-format tools require you to know
   the format up front.
3. **Truecolor + OSC-8 hyperlinks + width-aware tables in one
   pass.** Tables aren't just colored ASCII — they reflow to
   the terminal width, truncate cells with proper ellipsis,
   right-align numerics, and emit OSC-8 hyperlinks for any URL
   cell so modern terminals make them clickable.

For an LLM-CLI workflow, `rich-cli` is the **inspect-the-output
step**: an agent emits a Markdown report or a JSON tool-call
result, the human runs `agent --output report.md && rich
report.md` (or pipes the JSON straight in) and gets a readable
view without leaving the terminal. It's the inverse of
[`gum`](../gum/)-style input prompts — `gum` collects
keystrokes from the human, `rich` renders bytes for the human.

## Vs Already Cataloged

- **Vs [`bat`](../bat/):** `bat` is the *cat replacement*
  specialised for source code + Markdown, with paging and git
  integration. `rich-cli` covers a wider format set (CSV, JSON,
  inline images) and ships a unified theme, but is not a `cat`
  drop-in (no paging, no `--diff`, no git gutter). Use `bat`
  for "show me this file the way my editor does"; use `rich`
  for "render this arbitrary file or stdin pretty".
- **Vs [`glow`](../glow/):** `glow` is Markdown-only, with a
  TUI mode for browsing local + remote Markdown. `rich-cli` is
  one-shot rendering across many formats. For a terminal Markdown
  *reader* with paging, pick `glow`; for "pretty-print this one
  file and exit", pick `rich`.
- **Vs `jq` + `bat` + `csvlook` + `chafa`:** That stack works,
  but you maintain four configs and four theme files, and the
  invocation depends on the file type. `rich-cli` collapses the
  stack to one binary at the cost of "not the best at any single
  format" — `jq` is still the right answer for JSON *querying*
  (`rich` only renders), and `chafa` does sixel / kitty graphics
  far better than unicode half-blocks.
- **Vs [`hexyl`](../hexyl/):** Different domain — `hexyl` is for
  binary files (hex + ASCII gutter with byte-class colouring);
  `rich` refuses non-text and falls back to image preview only
  for known image formats.

## Caveats

- **Python startup tax.** First invocation in a fresh shell
  takes ~150–250 ms to import `rich` + Pygments + click. For a
  hot-loop of "render thousands of tiny files" this is the
  bottleneck; for interactive single-file viewing it's
  invisible. `bat` (Rust) starts in <10 ms.
- **Image rendering is unicode-block only.** No sixel, no kitty
  graphics protocol, no iTerm2 inline images. The half-block
  trick works in every terminal but caps useful resolution at
  ~2× the column count. For high-fidelity terminal images,
  reach for `chafa` or `kitty +kitten icat`.
- **No paging.** Output goes straight to stdout — pipe to
  `less -R` if you need pager behaviour. This is deliberate
  (composability), but it's a friction point if you're used to
  `bat`'s built-in pager.
- **Markdown rendering is `rich`-flavoured, not CommonMark-
  perfect.** Nested lists, complex HTML inlines, and footnotes
  may render approximately. For strict spec compliance run the
  Markdown through `pandoc` first.
- **Tag cadence is slow.** v1.8.1 (mid-2022) is the latest
  release; the project rides on top of `rich` itself, which is
  actively maintained. Bug-fix cadence is "as needed", and the
  feature surface is intentionally frozen.
