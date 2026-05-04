# lilypond

> **Programmable music engraving system** — text-driven musical
> notation compiler: write a `.ly` source file in LilyPond's
> domain-specific language (`\relative c' { c4 d e f | g2 c, }`)
> and compile it to publication-quality engraved sheet music as
> PDF / SVG / PostScript / PNG, with MIDI playback as a side
> output, font-correct at 1200 dpi, fully scriptable from a
> Makefile or CI — pinned to **v2.26.0** (released 2026-04-21,
> [LICENSE](https://github.com/lilypond/lilypond/blob/v2.26.0/LICENSE),
> SPDX `GPL-3.0-or-later`).

Source: <https://github.com/lilypond/lilypond>
Upstream homepage: <https://lilypond.org>

## TL;DR

The music notation space is split across two camps:
**WYSIWYG GUIs** (Finale — defunct since 2024, Sibelius — Avid,
Dorico — Steinberg, MuseScore — open source GPL) where you click
notes onto a staff, and **text-source engravers** where you
write a source file and compile it. LilyPond is the canonical
text-source camp: the input language is small (notes are
letters, durations are numbers, slurs are parens, dynamics are
backslash commands), the layout engine is the work of three
decades of engraving research, the output is indistinguishable
from hand-engraved 19th-century editions at print resolution.
Diff-able sources, version-controllable scores, programmatic
generation (Scheme `\override` blocks), and reproducible builds
are all native — exactly the properties a GUI editor cannot give
you. Pair with `frescobaldi` (GUI editor that drives lilypond
in the background) if you want both, but the CLI alone is
sufficient.

## Install

```bash
# macOS
brew install lilypond

# Debian / Ubuntu
sudo apt install lilypond

# Arch
sudo pacman -S lilypond

# Fedora
sudo dnf install lilypond

# Pre-built binaries (Linux x86_64 / aarch64, macOS arm64 /
# x86_64, Windows x64) for v2.26.0 live at:
#   https://github.com/lilypond/lilypond/releases/tag/v2.26.0
# Or upstream:
#   https://lilypond.org/download.html
```

Hard prereqs: none beyond a working text editor. The binary is
self-contained (vendored Guile Scheme runtime, vendored Pango
text shaper, vendored fonts). PDF/SVG output requires no
external `latex` or `dvipdf` toolchain.

## Common invocations

```bash
# Compile a single .ly file to PDF + MIDI
lilypond song.ly                 # produces song.pdf + song.midi

# SVG output (one file per page)
lilypond -dbackend=svg song.ly

# PNG output at 300 dpi (for web embeds)
lilypond -dbackend=eps --png -dresolution=300 song.ly

# Skip MIDI generation
lilypond -dno-include-book-title-preview --no-midi song.ly

# Convert a v2.20 source forward to v2.26 syntax (idempotent)
convert-ly --edit old-song.ly

# Engrave a folder of scores in parallel via xargs
ls *.ly | xargs -n1 -P4 lilypond
```

Minimal `song.ly`:

```lilypond
\version "2.26.0"
\header { title = "Hello, world" composer = "Anonymous" }
\score {
  \relative c' { c4 d e f | g2 c, | }
  \layout { } \midi { }
}
```

## Why orthogonal to existing zoo

The zoo has document compilers (`typst`, `pandoc`,
`asciidoctor`), audio file utilities (`sox`, `beets`), and
typesetting LSP work (`tinymist`, `marksman`) but **no music
notation engraver of any kind** — there is no other path in
this catalog from "I want a PDF score" to a printed PDF score.
The closest sibling is `typst` (general document compiler), but
typst has no music-engraving primitives: ledger lines, beaming
rules, slur curves, automatic accidentals, cautionary
accidentals, and historical engraving conventions are
LilyPond-specific.

## Caveats

- GPL-3.0-or-later: distributing modified builds requires source
  release. Embedding LilyPond as a *backend* in another product
  is fine when the product itself is GPL-compatible; embedding
  in proprietary software is not — call out to the binary as a
  separate process if you need looser license terms downstream.
- Compile-time is non-trivial: a single-page lead sheet
  compiles in ~1-3 s, a 200-page orchestral score with custom
  Scheme overrides can take 30-90 s on modern hardware. CI
  benefits from caching the compiled PDF when the source hash
  is unchanged.
- The DSL has a learning curve: notes-and-rhythms basics take
  an hour, fine engraving control (`\override Score.BarNumber`,
  Scheme `\with-color`, custom paper sizes, polyrhythmic
  beaming) is a multi-week investment. The official
  *Notation Reference* PDF is ~700 pages.
- `convert-ly` handles forward-compatibility from older
  versions, but Scheme `\override` blocks written against
  v2.18 internal grob names sometimes still need manual
  fix-up after `convert-ly` runs.
- For interactive editing prefer `frescobaldi` (Qt GUI that
  shells out to `lilypond` for compile + reload) or run
  `lilypond --loop` patterns from your editor's build hook.
- MIDI output is correct-pitch + correct-rhythm only, with no
  expressive performance — for realistic playback, pipe the
  generated `.midi` through a soundfont synth (`fluidsynth`).
