# mdcat

- **Repo:** https://github.com/swsnr/mdcat
- **Version:** v2.7.1 (latest stable, 2025)
- **License:** MPL-2.0 ([LICENSE](https://github.com/swsnr/mdcat/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install mdcat` · `cargo install --locked mdcat` · `pacman -S mdcat` · static binaries on the GitHub release page

## What it does

`mdcat` is `cat` for Markdown: it takes a `.md` file (or stdin) and renders
it to ANSI in your terminal with proper heading styles, syntax-highlighted
fenced code blocks (via `syntect`, the same engine `bat` uses), real
hyperlinks via OSC 8 escape sequences (so `cmd-click` works in iTerm2,
WezTerm, kitty, Ghostty, and modern GNOME Terminal), and **inline images**
via the kitty / iTerm2 / WezTerm / Sixel image protocols when your terminal
supports them. There is no pager, no interactive mode, no TOC sidebar —
it is a one-shot pretty-printer that exits, which is exactly what you want
inside a pipeline or a `less`-style invocation (`mdcat README.md | less
-R`). Tables render as aligned text with box-drawing characters; task
lists become checkbox glyphs; block quotes get a left border; rules become
horizontal lines that span the terminal width. Companion binary `mdless`
ships in the same release and pipes the output through your `$PAGER`
automatically. Configuration lives in a small TOML file at
`$XDG_CONFIG_HOME/mdcat/mdcat.toml` and lets you pin a syntax theme, a
default terminal type (handy for tmux which lies about capabilities), and
HTTP timeout / size limits for fetched images.

## When to pick it / when not to

Pick `mdcat` whenever you `cat README.md` and immediately wish you had not.
The right reflex on any unfamiliar repo is `mdcat README.md` — headings,
code blocks, and links all become legible without leaving the terminal,
and on a kitty / WezTerm / iTerm2 setup the architecture diagrams render
inline instead of showing as `![diagram](docs/arch.png)`. It is the right
tool for reading release notes (`gh release view --json body -q .body |
mdcat`), for previewing your own Markdown before pushing
(`mdcat CHANGELOG.md`), for piping `glow`-style output without `glow`'s
heavier UI, and for any agent / script that wants to surface formatted
output to a human in a terminal. Pair it with [`bat`](../bat/) for source
code (mdcat for prose, bat for code), with [`glow`](../glow/) when you
want a TUI with scroll / search / file picker rather than a one-shot
renderer, and with [`presenterm`](../presenterm/) when the Markdown is
actually slides.

Skip it for terminals without true-color or OSC-8 support — output still
works but the polish (real links, inline images, 24-bit syntax themes)
degrades to plain ANSI. Skip it for very long documents where you want
search / scroll / TOC — that is [`glow`](../glow/)'s job, not mdcat's.
Skip it for GitHub-Flavored-Markdown extensions that require a live
renderer (Mermaid diagrams, GitHub-specific alert blocks, footnote
back-references) — mdcat handles CommonMark + tables + task lists +
strikethrough, but Mermaid blocks render as their source text. Skip it
in CI logs that will be read in a non-color-capable viewer (some CI web
UIs strip ANSI inconsistently); just keep the raw Markdown there. And
note: image rendering requires the terminal to actually speak the kitty
or iTerm2 image protocol — over a plain SSH session into a tmux running
inside a basic xterm, you will get text-only output, which is correct
behaviour but sometimes surprising.

## Example invocations

```bash
# Pretty-print a README to the terminal
mdcat README.md

# Read it through your pager (companion binary, ships in the same release)
mdless CHANGELOG.md

# Pipe from another tool (gh, jq, curl)
gh release view --json body -q .body | mdcat

# Render a remote Markdown file (mdcat fetches it itself)
mdcat https://raw.githubusercontent.com/rust-lang/rust/master/README.md

# Force a specific terminal capability set (useful inside tmux)
mdcat --terminal=iterm2 README.md

# Disable inline image fetching (offline / sandbox / CI)
mdcat --local-only README.md

# Pin a syntax-highlight theme for code blocks
mdcat --theme=Solarized\ \(dark\) docs/architecture.md

# Use as a git diff pager for .md files via .gitattributes + a custom driver
git config diff.mdcat.textconv "mdcat --local-only"
echo "*.md diff=mdcat" >> .gitattributes
```
