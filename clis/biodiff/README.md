# biodiff

> **A binary diff viewer that aligns two files
> using bioinformatics sequence-alignment
> algorithms (WFA2 / Needleman-Wunsch), then
> shows the result as a side-by-side hex view
> with insertions and deletions properly
> gapped** — instead of `cmp` / `vbindiff` style
> "everything after byte N is red because one
> byte was inserted". Pinned to **v1.2.1**
> ([LICENSE](https://github.com/8051Enthusiast/biodiff/blob/main/LICENSE),
> MIT).

Source: <https://github.com/8051Enthusiast/biodiff>

## TL;DR

Diffing binaries with line-oriented tools is
useless. Diffing them with byte-by-byte tools
(`cmp`, `xxd | diff`, `vbindiff`) breaks the
moment a single byte is *inserted* — every
subsequent offset shifts and the whole rest of
the file shows as different. Reverse-engineers,
firmware hackers, ROM hackers, and anyone
comparing two slightly-different builds of the
same binary need *alignment*: the same insight
DNA sequencing relies on. `biodiff` borrows the
WFA2 (wavefront) and Needleman-Wunsch
algorithms from bioinformatics and applies them
to byte streams, so a 3-byte insert near the
top of a 4 MB ELF stays a 3-byte insert — the
remaining 99.99% of the file lines up cleanly.

## Install

```bash
# cargo (recommended; needs Rust ≥ 1.70)
cargo install biodiff

# Homebrew
brew install biodiff

# Pre-built binary (Linux / macOS / Windows)
curl -LO https://github.com/8051Enthusiast/biodiff/releases/download/v1.2.1/biodiff-macos-1.2.1.zip
unzip biodiff-macos-1.2.1.zip
sudo install biodiff /usr/local/bin/

biodiff --version    # biodiff 1.2.1
```

## Use it for

```bash
# Compare two firmware blobs
biodiff old-firmware.bin new-firmware.bin

# Compare two builds of the same ELF
biodiff build-a/app build-b/app

# Inside the TUI:
#   F1            help
#   space / 0..9  cycle alignment algorithm (WFA2 is fastest)
#   /             search hex pattern
#   r             realign from cursor
```

The two files scroll in lock-step under the
chosen alignment; gapped regions are visually
marked, so a 4-byte patch in the middle of a
2 MB binary shows up as exactly 4 highlighted
bytes — not 2 MB of red.

## Why include it in a CLI catalog

1. **It's the only widely-used hex differ that
   does *real* alignment.** `vbindiff` and
   `dhex` show two files side-by-side but
   assume same offsets; `radiff2` (radare2)
   does block-level diff but isn't a viewer.
   `biodiff` is purpose-built for "what
   actually changed between these two
   binaries".
2. **WFA2 is genuinely fast.** On multi-MB
   binaries, the wavefront algorithm finishes
   in seconds where naive global alignment
   would take minutes — making `biodiff`
   usable for whole-file firmware diffs, not
   just toy examples.
3. **Niche but irreplaceable.** Firmware
   updates, packed-binary deltas, save-game
   editing, ROM hacking, malware family
   triage, A/B build comparison — anywhere
   you'd say "I wish `diff` understood
   bytes", `biodiff` is the answer.

## Caveats

- Alignment over very large files (hundreds
  of MB+) can run out of memory; pre-slice or
  diff sections individually.
- TUI-only — no `--patch` / machine-readable
  output. Pair with `radare2` /
  [`bingrep`](../bingrep/) /
  [`hexyl`](../hexyl/) for scripted analysis.
- For text-shaped binaries (logs, JSON
  blobs), `delta` / `difftastic` are the
  right tools — `biodiff` shines on opaque
  byte streams.
