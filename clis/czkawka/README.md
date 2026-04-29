# czkawka

> **A Rust-implemented multi-engine duplicate / waste-file finder with a
> CLI (`czkawka_cli`) and a GTK4 GUI (`czkawka_gui`) sharing one core
> engine — finds duplicate files (XXH3 / Blake3 / CRC32 / SHA256 hash
> ladder), similar images (perceptual hashes: Mean / Gradient / DCT /
> Blockhash / VertGradient at configurable Hamming-distance thresholds),
> similar videos (frame-fingerprint), big files, empty files, empty
> dirs, broken symlinks, broken music files, similar music (audiotag
> vs fingerprint modes), invalid extensions, temporary files, and bad
> symlinks — single-pass scan, one Rust binary, no Python / Node /
> JVM runtime.** Pinned to **v11.0.1**
> ([LICENSE](https://github.com/qarmin/czkawka/blob/master/LICENSE_MIT_EVERYTHING_OUTSIDE_ANY_CARGO_APP_LIBRARY),
> MIT for code; icons under
> [LICENSE_CC_BY_4_ICONS](https://github.com/qarmin/czkawka/blob/master/LICENSE_CC_BY_4_ICONS),
> CC-BY-4.0).

Source: <https://github.com/qarmin/czkawka>

## TL;DR

`czkawka` (Polish for "hiccup") is what you reach for when a 4 TB
photo / project / download dir has accumulated years of duplicates,
near-duplicates, and resized variants and the existing options are
unsatisfying — `fdupes` only finds *byte-exact* duplicates and is
slow on huge trees, GUI tools (Gemini, dupeGuru) are macOS-only or
Python-Qt-heavy, and rolling your own `find -size` + `xxhsum` +
`sort -u` pipeline misses the *similar-but-not-identical* case
(same photo at 4032×3024 and 1920×1080, same MP3 at 320 kbps and
192 kbps). `czkawka` ships one Rust binary that handles all of
those passes through one config-driven scan: a hash ladder for
exact dupes (cheap XXH3 first, then re-hash collisions with a
stronger algo), perceptual image hashing for visually-similar
photos at configurable similarity thresholds, frame-sampling video
fingerprints, and audio fingerprint or tag-comparison modes for
music. Output is one TSV / JSON / plain-text report per scanner
mode, ready to pipe into `xargs rm` after human review (or its
sibling `krokiet` GUI for a click-driven review pane).

## Install

```bash
# Homebrew (macOS / Linux) — installs CLI binary as `czkawka_cli`
brew install czkawka

# Linux package managers
apt install czkawka                  # Debian / Ubuntu (>= 24.04)
dnf install czkawka                  # Fedora
pacman -S czkawka-cli czkawka-gui    # Arch (split package)
flatpak install com.github.qarmin.czkawka  # Flatpak (GUI bundle)

# Cargo (any platform with Rust >=1.85)
cargo install czkawka_cli --version 11.0.1
cargo install czkawka_gui --version 11.0.1     # optional GTK4 GUI

# Static prebuilt binary (Linux x86_64 / macOS / Windows)
# https://github.com/qarmin/czkawka/releases/tag/11.0.1

# verify
czkawka_cli --version                # 11.0.1
```

## License

**MIT** (code) — see
[LICENSE_MIT_EVERYTHING_OUTSIDE_ANY_CARGO_APP_LIBRARY](https://github.com/qarmin/czkawka/blob/master/LICENSE_MIT_EVERYTHING_OUTSIDE_ANY_CARGO_APP_LIBRARY).
Icons are CC-BY-4.0 — see
[LICENSE_CC_BY_4_ICONS](https://github.com/qarmin/czkawka/blob/master/LICENSE_CC_BY_4_ICONS).
For redistribution: the binary itself only requires MIT-style
attribution; only matters if you ship the GUI's icon set in
another product.

## One Concrete Example

```bash
# 1. exact-byte duplicates in ~/Pictures, prefer keeping the oldest
#    file in each duplicate group, write report and act with --delete-method
czkawka_cli dup -d ~/Pictures \
    -m 100KB -x jpg,jpeg,png,heic \
    --search-method hash --hash-type xxh3 \
    --delete-method aen --file-to-save dupes.txt

# 2. visually-similar photos (perceptual hash, 16-bit DCT,
#    similarity ~"Small" tolerance) — emit a report, don't touch files
czkawka_cli image -d ~/Pictures \
    --hash-alg dct --hash-size 16 --similarity-preset small \
    --file-to-save similar-photos.txt

# 3. similar music by fingerprint (chromaprint-style); group near-duplicates
czkawka_cli music -d ~/Music \
    --search-method fingerprint --maximum-difference 2.0 \
    --minimal-fragment-duration 5.0 \
    --file-to-save similar-music.txt

# 4. big files (top 50 by size) — useful pre-cleanup survey
czkawka_cli big -d / --excluded-directories /proc,/sys,/dev \
    --number-of-files 50

# 5. empty dirs + broken symlinks in one sweep
czkawka_cli empty-folders -d ~/Projects --delete-folders
czkawka_cli symlinks -d ~/Projects --delete-files

# 6. compose with shell — review then delete
czkawka_cli dup -d ~/Downloads --search-method hash \
    --file-to-save - | tee /tmp/dupes.txt
# inspect, then:
awk '/^\// && NR>1 {print}' /tmp/dupes.txt | \
    fzf -m --preview 'ls -lh {}' | xargs -d '\n' rm -i
```

## Niche It Fills

**Replace `fdupes` + a per-media-type ad-hoc tool stack with one
Rust binary that does exact, perceptual, and fingerprint-based
de-duplication across files, images, videos, and audio in one
scan.** The shape competitors fill is fragmented: `fdupes` /
`rdfind` / `jdupes` only find byte-exact duplicates; `findimagedupes`
/ `imgdupes` cover perceptual-image-hash but are Perl / Python with
heavy install footprints; `dupeGuru` is GUI-only and Python-Qt;
`Gemini` is macOS-only and proprietary. `czkawka` collapses all of
those into one config-driven binary with a CLI you can pipe and a
GTK4 GUI for the human-review step, runs at native speed on huge
trees (multi-threaded scan, content-addressable cache between runs),
and is small enough to drop into a CI job that flags duplicate
artifacts before a release tarball ships.

## Why use it

Three things `czkawka` does that `fdupes` does not:

1. **Perceptual + fingerprint modes, not just byte-exact.** The
   `image` scanner hashes via Mean / Gradient / DCT / Blockhash /
   VertGradient at configurable hash sizes (8 / 16 / 32 / 64 bits)
   and Hamming-distance similarity presets (Minimal / VerySmall /
   Small / Medium / High), so the same photo at 4032×3024 and
   1920×1080 lands in the same group. The `music` scanner has a
   `fingerprint` mode (audio chromaprint-style) for "same song,
   different bitrate / encoder" and a `tags` mode for ID3
   metadata-driven duplicate detection. The `videos` scanner
   samples frames and computes a perceptual fingerprint per video.
   `fdupes` and friends miss all of these because they only compare
   bytes.
2. **Hash ladder + on-disk cache for fast re-scans.** First pass is
   cheap XXH3; collisions get re-hashed with a stronger algo
   (Blake3 / SHA256 / CRC32) before claiming "duplicate," so a
   billion small files are not all SHA256-hashed up front. Hashes
   are persisted in a per-tool cache (`~/.cache/czkawka/`) keyed by
   path + size + mtime, so a daily re-scan of the same photo
   library does not re-read every byte — only changed files are
   re-hashed. On a 4 TB photo dir the second scan is typically
   10-50× faster than the first.
3. **One binary covers eight scanner modes — and they share config.**
   `dup` (duplicates), `empty-folders`, `empty-files`, `big`,
   `temporary`, `image` (similar images), `music` (similar music),
   `symlinks` (broken), `same-music`, `videos` (similar videos),
   `extension` (mismatched ext vs MIME), `broken` (broken music
   files). Same `--excluded-directories` / `--allowed-extensions`
   / `--minimal-file-size` / `--reference-directories` /
   `--file-to-save` / `--delete-method` flags work across all of
   them, so one `czkawka_cli.toml` config file or one shell
   wrapper covers a whole cleanup workflow.

For a "clean up a 4 TB photo dir before backing up" workflow,
`czkawka_cli dup` then `czkawka_cli image --similarity-preset small`
then `czkawka_cli big -n 100` is three commands and one binary,
producing three TSV reports you walk in `fzf` or load into the GUI
for the irreversible delete step.

## Vs Already Cataloged

- **Vs [`dust`](../dust/) / [`procs`](../procs/) /
  [`bandwhich`](../bandwhich/):** different observational layer.
  Those tell you *where the disk / memory / network is going right
  now*; `czkawka` tells you *which specific files to delete to get
  it back*. Common sequence: `dust ~` highlights the fattest
  subtree, then `czkawka_cli dup -d <subtree>` figures out how
  much of it is recoverable duplicate weight.
- **Vs [`ouch`](../ouch/):** orthogonal. `ouch` is a universal
  compressor; `czkawka` is a duplicate finder. They compose: run
  `czkawka` first to remove duplicates, then `ouch compress` the
  cleaner result for archival.
- **Vs `fdupes` / `rdfind` / `jdupes` (not cataloged):** czkawka
  is a strict superset for the duplicate-finding case (XXH3 ladder
  matches their speed on byte-exact, then adds perceptual /
  fingerprint modes the older tools cannot do). Pick `fdupes` only
  if "single C binary, packaged in every distro since 2003" is the
  dealbreaker.
- **Vs `dupeGuru` (not cataloged):** czkawka is the CLI-first
  answer; dupeGuru is the GUI-only Python-Qt answer. czkawka also
  ships a GTK4 GUI (`czkawka_gui`) and a Slint-based GUI
  (`krokiet`) for the click-driven review step, so you can use the
  CLI in CI / cron and the GUI for ad-hoc cleanup, against the
  same scan reports.

## Caveats

- **GUI is GTK4 — heavyweight on macOS.** The CLI (`czkawka_cli`)
  is fine on macOS / Linux / Windows. The GTK4 GUI works but the
  install footprint on macOS is large (GTK4 runtime via Homebrew).
  Consider the Slint-based `krokiet` GUI from the same project for
  a lighter native option, or stick to the CLI + `fzf` workflow.
- **`--delete-method aen` (all except newest) is destructive and
  per-group.** czkawka deletes whichever file *each duplicate
  group* identifies as "not the kept one"; use
  `--reference-directories` to force a specific tree to be the
  source of truth (no files in that tree get deleted), and prefer
  `--file-to-save` first then `xargs rm -i` for any high-stakes
  cleanup. The CLI has no global "preview the deletions" flag —
  the report-then-rm workflow is the way.
- **Perceptual image hashing is heuristic.** Similarity presets
  trade false positives against false negatives — `Small` catches
  most resized / re-encoded photos, `High` catches near-duplicates
  with crops / colour shifts but also clusters unrelated photos
  with similar composition. Always review the report before
  deleting; the GUI is much faster than the CLI for this step.
- **Audio `fingerprint` mode requires Chromaprint.** The
  fingerprint scanner uses libchromaprint via `rusty_chromaprint`;
  on Linux this needs the `libchromaprint` package
  (apt install libchromaprint1, or built-in to the
  Cargo build). The `tags` mode has no native dep. If a
  fingerprint scan returns zero results check the runtime dep
  before assuming there are no duplicates.
- **Cache invalidation is path + size + mtime, not content.** If a
  build process touches mtime without changing content, czkawka
  re-hashes the file — wasted work but correct. If a process
  changes content while preserving size + mtime (rare but
  possible with mmap-write workloads), czkawka uses the stale
  cached hash — incorrect. Delete `~/.cache/czkawka/` to force a
  full re-scan if you suspect cache poisoning.
- **MIT-only license on the code, CC-BY-4.0 on the icons.** The
  binary is fine to redistribute under MIT; only the GUI's icon
  set carries the CC-BY attribution requirement, which only
  matters if you ship those icons inside another product.
