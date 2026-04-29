# fselect

## Overview
`fselect` lets you find files using SQL-like `SELECT ... FROM ... WHERE ...` queries. Instead of memorizing `find` flags, you write expressive predicates against file attributes (name, size, mtime, mime, exif, hash, line count, etc.) and aggregate them with `GROUP BY` / `ORDER BY` / `LIMIT`.

## Repo URL
https://github.com/jhspetersson/fselect

## Version
`0.10.0` (released 2026-04-10)

## License
- SPDX: `Apache-2.0 OR MIT` (dual-licensed)
- License files in upstream repo:
  - `LICENSE-APACHE`
  - `LICENSE-MIT`

## Why it matters
- One mental model (SQL) replaces a wall of `find -type f -name ... -mtime ... -exec ...` incantations.
- Built-in functions for hashing, EXIF, ID3, MP3, archive contents, and image dimensions — things `find` cannot do natively.
- Output as `csv`, `json`, `html`, or human tables, so it composes cleanly into pipelines.
- Single static Rust binary, ~4.4k stars, actively maintained.

## Install
```sh
# macOS
brew install fselect

# Cargo
cargo install fselect

# Binary releases
# https://github.com/jhspetersson/fselect/releases
```

## Quick example
```sh
# 10 largest *.log files modified in the last 7 days, under /var/log
fselect "size, path FROM /var/log WHERE name = '*.log' AND modified gt today - 7d \
         ORDER BY size DESC LIMIT 10"

# Find duplicate JPEGs by sha256
fselect "sha256, path FROM ~/Pictures WHERE ext = 'jpg' GROUP BY sha256 HAVING count(*) > 1"
```

## Comparison context in this zoo
- Replaces / augments classic `find` (no entry) and works alongside zoo entries:
  - `fd` — fast `find` clone, regex/glob centric, no SQL or content-aware predicates.
  - `fclones` — focused exclusively on duplicate detection; `fselect` is more general.
  - `ast-grep` — structural search over source code ASTs; complementary, not overlapping.
  - `ripgrep` — content search inside files; `fselect` filters which files, then you pipe into `rg`.
