# bat

> **`cat(1)` clone with syntax highlighting and Git integration** —
> a single Rust binary that reads files (or stdin) and prints them
> with language-aware colour, line numbers, paged output for long
> files, and an inline Git diff gutter showing added / modified /
> deleted lines against `HEAD`. Pinned to **v0.26.1** (commit
> `979ba22628bc9d8171f2cffca2bd5c90c9fc0a9e`,
> [LICENSE-MIT](https://github.com/sharkdp/bat/blob/master/LICENSE-MIT),
> dual MIT / Apache-2.0).

Source: <https://github.com/sharkdp/bat>

## TL;DR

`bat` is the answer when `cat file.py` dumps a wall of monochrome
text and you have to scroll back, find the function you wanted,
and squint. Same call shape (`bat file1 file2 …`), same pipe
behaviour (`bat file | grep …` works), but the output is
syntax-highlighted for ~200 languages out of the box, paged
through `less` automatically when it does not fit, prefixed with
line numbers, and decorated with a Git gutter (`+` / `~` / `-`)
when the file is in a repo. It also doubles as a colourising
pretty-printer in pipes: `curl -s api.example.com/x | bat -l json`
gives you the same coloured JSON view without piping through `jq`
just for the syntax. Designed to be a drop-in: when stdout is not
a TTY (e.g. inside a pipe), `bat` prints plain text by default
and behaves exactly like `cat`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install bat

# Cargo
cargo install --locked bat

# Linux package managers
# Arch: pacman -S bat
# Debian / Ubuntu: apt install bat   # binary may be named `batcat`
# Fedora: dnf install bat
# Nix: nix-env -iA nixpkgs.bat

# Windows
# scoop install bat
# choco install bat
# winget install sharkdp.bat

# from a release tarball (any OS)
curl -Lo bat.tar.gz "https://github.com/sharkdp/bat/releases/download/v0.26.1/bat-v0.26.1-aarch64-apple-darwin.tar.gz"
tar xf bat.tar.gz --strip-components=1
sudo install bat /usr/local/bin/

# verify
bat --version    # bat 0.26.1
```

On Debian / Ubuntu the binary is installed as `batcat` to avoid a
naming collision with an existing `bat` package; either alias it
(`alias bat=batcat`) or symlink (`ln -s "$(which batcat)" ~/.local/bin/bat`).
First run will write a small config to `~/.config/bat/`; nothing
required to make it useful.

## License

Dual MIT / Apache-2.0 — see
[LICENSE-MIT](https://github.com/sharkdp/bat/blob/master/LICENSE-MIT)
and [LICENSE-APACHE](https://github.com/sharkdp/bat/blob/master/LICENSE-APACHE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. read a file with syntax highlight + line numbers + Git gutter
bat src/main.rs
# │ ~ │ 12 │ fn main() {
# │ + │ 13 │     println!("hello");          ← line added since HEAD
# │   │ 14 │ }

# 2. range slice (great for grep / lint output references)
bat --line-range 40:80 src/parser.rs

# 3. show only changed lines around diffs (compact review)
bat --diff src/lib.rs

# 4. force a language for stdin (no extension hint)
curl -s https://api.github.com/repos/sharkdp/bat | bat -l json

# 5. plain mode — strips colour, line numbers, paging (useful in pipes
#    if you really want bat behaviour but not bat decoration)
bat -pp script.sh   # = --plain --plain (no decorations, no paging)

# 6. concatenate many files into one paged view
bat src/*.rs

# 7. as a $PAGER replacement (man pages get syntax highlighting)
export MANPAGER="sh -c 'col -bx | bat -l man -p'"
man rsync   # now syntax-coloured

# 8. as the highlighter inside fzf preview windows
fzf --preview 'bat --color=always --style=numbers --line-range=:200 {}'
```

## Niche It Fills

**The TTY-aware default file viewer.** For day-to-day reading of
source files, configs, log files, JSON / YAML / TOML, `cat` is
intentionally minimal and `less` is intentionally modal; `bat`
sits in the middle: paged when it needs to be, colourised when
the terminal can take it, plain when piped, and shaped exactly
like `cat` so muscle memory transfers. The "TTY-detect, behave
like `cat` in pipes" default is the load-bearing piece — you can
alias `cat=bat` and not break shell scripts.

## Why use it

Three things `bat` does that `cat` does not, that pay back the
switching cost:

1. **Syntax highlight by language, not by file size.** ~200
   grammars (Sublime Text `.sublime-syntax` format) cover every
   common config / source / data file; the language is detected
   from extension first, shebang second, content sniffer third.
   For `.env` / `Dockerfile` / `nginx.conf` / `Cargo.toml` /
   `compose.yaml` this means the structure is visible at a
   glance, not after `:set syntax=…` in a separate editor.
2. **Git diff gutter is on by default in a repo.** `+` / `~` /
   `-` markers in the left margin show what changed since `HEAD`
   without running `git diff` in another pane; combined with line
   numbers it makes "what did I touch in this file" a one-command
   answer.
3. **Pager + line numbers + grid as one default.** Files longer
   than the terminal automatically open in `less -R`; line
   numbers on the left mean grep / linter output ("error at
   `lib.rs:412`") is one-keystroke navigable; the box-drawing
   grid separates header / body / footer cleanly without the
   noise of a real text editor.

For an LLM-CLI workflow where you are pasting a file region into
a chat, `bat -p --line-range 100:200 src/x.rs | pbcopy` gives you
plain (no colour codes), line-bounded text — the same shape an
agent wants, with the slice ergonomics `cat` cannot do without
piping through `sed`.

## Vs Already Cataloged

- **Vs [`delta`](../delta/):** orthogonal — `delta` is a
  syntax-highlighting *diff* viewer (`git diff` / `git log -p` /
  `git show` get richer side-by-side or unified colour output);
  `bat` is a syntax-highlighting *file* viewer for whole files,
  slices, and stdin pipes. They are designed to be used together:
  `delta` for "what changed", `bat` for "what does the file look
  like now". Both ship as Rust binaries with the same Sublime-
  syntax engine and feel like siblings.
- **Vs [`glow`](../glow/) (not cataloged) / `mdcat` (not
  cataloged):** orthogonal — those render Markdown to terminal as
  *prose* (headings styled, links unfurled, code blocks
  highlighted). `bat` shows Markdown as *source code* — useful
  when you want to see exactly the bytes, not the rendered view.
  Use `glow README.md` to read the README; use `bat README.md`
  to inspect the raw markdown when you are about to edit it.
- **Vs `less` / `cat` (POSIX):** `cat` is the right answer for
  pure stdout pipes and tiny files where colour is noise; `less`
  is the right answer when you want full pager modes (`/search`,
  `n`/`N`, mark-and-jump, follow-with-`F`). `bat` is the right
  answer for the 80% middle: read this file, with structure
  visible, scroll if it is long, plain-text behaviour if I pipe
  it. You keep `less` for log following and `cat` for scripts.

## Caveats

- **Cold start is slower than `cat`.** A single `bat foo.txt`
  pays a few milliseconds to load syntax / theme tables. For a
  human reading a file this is invisible; for a tight shell
  loop (`for f in *.txt; do bat -p "$f"; done` over thousands of
  files) it matters — use `cat` or `bat --no-config -p` in that
  case.
- **Theme picking is a rabbit hole.** Default theme is
  `Monokai Extended` and assumes a dark terminal; on a light
  terminal pass `--theme=GitHub` or set `BAT_THEME` in the
  environment. `bat --list-themes` previews each theme on a
  sample file; pin the choice in `~/.config/bat/config` to make
  it sticky.
- **Debian / Ubuntu rename to `batcat`.** Scripts that assume
  `bat` exists by name break on those distros; alias or symlink
  in CI images that target multiple OSes.
- **Git gutter respects `HEAD`, not the index.** Lines you have
  staged but not committed still show as added; lines you have
  unstaged but tracked show as modified. There is no "diff
  against staged" mode — for that, pipe `git diff --cached` to
  [`delta`](../delta/) instead.
- **Paging interacts with `less`'s exit behaviour.** On terminals
  where `less -R` exits clearing the screen, `bat foo` reads as
  "nothing happened"; set `BAT_PAGER='less -RFX'` (or
  `LESS=-RFX`) so short output stays printed and long output
  pages without screen-clearing.
- **Not a `cat` replacement for binary files.** `bat` detects
  binary content and refuses to dump it (you get a `[bat warning]
  Binary content` message); for hex dumps reach for `hexyl` (a
  sibling project) or plain `xxd`.
