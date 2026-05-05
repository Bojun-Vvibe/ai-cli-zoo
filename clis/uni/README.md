# uni

> **Query the Unicode database from the command line** — look up
> codepoints by name, character, hex, or category; search emojis
> by description, group, subgroup, skin-tone, or gender; and pipe
> the results back into the shell as plain text without leaving
> the terminal for a browser or a `python -c "import unicodedata"`
> one-liner. Pinned to **v2.9.0** (SPDX: `MIT`,
> [LICENSE](https://github.com/arp242/uni/blob/main/LICENSE)).

Source: <https://github.com/arp242/uni>

## TL;DR

`uni` is a single Go binary (~5 MB) that ships an embedded copy
of the Unicode Character Database and the Unicode Emoji data
files, then exposes it as five subcommands (`identify`, `search`,
`print`, `emoji`, `help`) with a uniform output schema (codepoint
in hex, decimal, UTF-8 bytes, name, category, block, script, …).
The killer property is **library-free, network-free, dependency-
free Unicode lookup at the shell prompt** — no `python -c`, no
`charmap.com`, no Emojipedia tab — so `uni s 'thinking face'`
returns `🤔` and `uni i 🤔` returns its full codepoint record in
under 10 ms.

## Install

```bash
# Go install
go install zgo.at/uni/v2@latest

# Homebrew
brew install uni

# Pre-built binary (Linux / macOS / Windows)
curl -LO "https://github.com/arp242/uni/releases/download/v2.9.0/uni-v2.9.0-linux-amd64.gz"
gunzip uni-v2.9.0-linux-amd64.gz
chmod +x uni-v2.9.0-linux-amd64
sudo install uni-v2.9.0-linux-amd64 /usr/local/bin/uni

# verify
uni -v   # uni v2.9.0
```

## License

MIT — see
[LICENSE](https://github.com/arp242/uni/blob/main/LICENSE).

## One Concrete Example

```bash
# 1. identify a character pasted from anywhere
uni identify 🤔
#   CPoint  Dec    UTF-8        Cat  Name
#   U+1F914 129300 f0 9f a4 94  So   THINKING FACE

# 2. search the UCD by name fragment
uni search 'snowman'
#   U+2603  ☃   So   SNOWMAN
#   U+26C4  ⛄  So   SNOWMAN WITHOUT SNOW
#   U+1F3C2 🏂  So   SNOWBOARDER

# 3. print a whole Unicode block
uni print 'Box Drawing'
#   U+2500  ─   So   BOX DRAWINGS LIGHT HORIZONTAL
#   U+2501  ━   So   BOX DRAWINGS HEAVY HORIZONTAL
#   ...

# 4. search emoji by description, restrict by group, apply tone
uni emoji -tone medium 'waving hand'
#   👋🏽  WAVING HAND: MEDIUM SKIN TONE  (people-body / hand-fingers-open)

# 5. machine-readable output for scripts
uni i -f all 🤔
#   cpoint    : U+1F914
#   dec       : 129300
#   utf8      : f0 9f a4 94
#   html      : &#x1F914;
#   xml       : &#129300;
#   json      : \uD83E\uDD14
#   name      : THINKING FACE
#   cat       : So (Symbol, Other)
#   block     : Supplemental Symbols and Pictographs
#   script    : Common
#   plane     : Supplementary Multilingual Plane

# 6. pipe a file's mystery character through identify
head -c 1 mystery.txt | uni identify -
```

## Niche It Fills

**The "what *is* this character" answer at the shell prompt.**
Three workflows it eats:
- "I pasted this string and one of the characters renders as a
  square — which one?" `uni i <(pbpaste)` lists every codepoint
  in the paste with name + category, finding the U+200B zero-
  width-space or the U+00A0 non-breaking-space hiding inside.
- "What's the snowman emoji's codepoint for this CSS rule?" —
  `uni s snowman` answers in a keystroke versus opening
  Emojipedia.
- "I need the Box Drawing block printed in a script's
  generated docs" — `uni p 'Box Drawing'` is the deterministic
  table, no manual transcription.

The closest catalog neighbor is `unicode` / `charinfo` / Python
`unicodedata` — those exist but are platform-specific, slower
to invoke, and lack the emoji-by-description search.

## Why use it

1. **Embedded UCD.** The Unicode database is *in the binary* —
   no network call, no system data file lookup, works on an
   air-gapped host the same as on a connected laptop.
2. **Five verbs cover everything.** `identify` (string →
   codepoints), `search` (name fragment → codepoints), `print`
   (block / category / script → codepoints), `emoji`
   (description / group / tone → emoji), `help`. Memorable,
   composable.
3. **Tone + gender modifiers for emoji.** `uni e -tone dark
   -gender man 'detective'` returns the exact composed
   ZWJ sequence — the right answer when a script needs to emit
   the inclusive variant programmatically.
4. **Tabular output by default, structured on demand.** Default
   columns align for human reading; `-f all` dumps every field
   for `awk` / `jq`-via-`jc` consumption.
5. **Stdin-aware.** `cat file | uni i -` identifies every
   codepoint in the input — the right shape for "audit this
   string for invisible characters."
6. **Tracks Unicode releases.** v2.x ships Unicode 16.0 + Emoji
   16.0 data; the version bump cadence matches the UTC's
   release cadence.

## Vs Already Cataloged

- **Vs [`figlet`](../figlet/) / [`boxes`](../boxes/):** Those
  *render* text into ASCII art. `uni` *introspects* characters'
  metadata. Different layers — they compose: `uni p 'Box
  Drawing' | head -1 | awk '{print $2}'` gives `figlet` the
  glyph to use as a frame.
- **Vs `iconv` / `recode`:** Those *transcode* between
  encodings. `uni` *describes* codepoints. Pick `iconv` when the
  bytes are wrong; pick `uni` when the codepoint is mystery.
- **Vs Python `unicodedata`:** `python3 -c "import unicodedata;
  print(unicodedata.name('🤔'))"` works but is 80 ms cold-start
  and has no emoji-by-description search; `uni i 🤔` is
  sub-10 ms.
- **Vs Emojipedia / unicode.org browse:** Browser tab, mouse,
  context switch. `uni e 'face palm'` is a keystroke at the
  prompt where you already are.

## Caveats

- **Not a font validator.** `uni` knows the codepoint exists in
  Unicode; it does not know whether your terminal font *renders*
  it. A correctly-named codepoint may still display as `□` if
  the active font lacks the glyph.
- **Embedded UCD ages.** When Unicode 17.0 ships, you need a
  `uni` upgrade to query the new codepoints. Pin the version
  in dotfiles for reproducibility.
- **Single-character emoji vs ZWJ sequences.** Composed emoji
  (skin-tone + gender + base) are returned as multi-codepoint
  ZWJ sequences — correct, but downstream code that assumes
  one codepoint per emoji (`len(s) == 1`) will be wrong. Use
  `uni e -f raw` for the bytes and treat the result as a string,
  not a character.
- **CLI flag rename in v2.** v1 used different flag names; v2
  is the supported branch. Check `uni help` if examples from old
  blog posts don't work.
