# sad

## Overview
`sad` ("Space Age seD") is an interactive batch find-and-replace CLI. You pipe a file list (e.g. from `fd` or `rg --files`), give it a pattern + replacement, and it shows you a unified diff per file with a TUI confirmation step before any write hits disk.

## Repo URL
https://github.com/ms-jpq/sad

## Version
`v0.4.32` (released 2025-02-03)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Why it matters
- Closes the gap between `sed -i` (fast but blind) and an editor's project-wide replace (interactive but not scriptable).
- Uses `git diff` / `delta` to render previews, so the output looks like a normal review diff.
- Supports both literal and regex patterns; safe-by-default — nothing is written until you confirm.
- ~2k stars, single Python entry point with a Rust-style UX.

## Install
```sh
# macOS
brew install sad

# Cargo (recommended for static binary)
cargo install sad

# Pre-built binaries
# https://github.com/ms-jpq/sad/releases
```

## Quick example
```sh
# Rename a symbol across a Rust project, with interactive confirmation
fd -t f -e rs | sad 'OldName' 'NewName'

# Regex mode, dry-run preview only (no commit)
fd -t f -e md | sad --pattern '\bv0\.9\.\d+\b' --replace 'v0.10.0' --commit
```

## Comparison context in this zoo
- Sits between several existing zoo entries:
  - `sd` — non-interactive `sed` replacement; `sad` adds the diff-confirmation TUI on top of the same idea.
  - `ast-grep` — structural, syntax-aware refactors; `sad` is line/regex level but works on any text.
  - `ripgrep` — finds matches; pair `rg --files-with-matches | sad ...` for a complete review-driven rewrite loop.
  - `delta` — `sad` reuses delta-style rendering for its preview pane.
