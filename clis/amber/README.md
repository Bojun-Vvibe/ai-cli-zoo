# amber

## What it does
A **code search and replace tool** in Rust that pairs a `ripgrep`-class
recursive search front (`ambs <pattern>`) with a transactional batch
replace front (`ambr <pattern> <replacement>`) operating on the same
match set. `ambr` opens every candidate hit in `$EDITOR` as a single
combined diff buffer, lets you delete or edit individual hunks before
the rewrite is committed, and writes all surviving edits in one pass —
so a project-wide rename is "find, prune the false positives in a real
editor, save, exit, done" rather than a manual `rg | xargs sed -i` loop
where you only learn afterwards which hits were spurious. Multibyte
text is handled correctly (UTF-8 / UTF-16 LE/BE / EUC-JP / Shift_JIS
auto-detected per file), `.gitignore` / `.ignore` / hidden files are
respected by default, regex is full-featured PCRE-style with backrefs
in the replacement, and parallelism is automatic across logical cores.
Single static binary, no runtime, no plugins.

## Why it's interesting
Different shape from `sed -i` (line-oriented, no preview, no false-
positive pruning, easy to corrupt files on a bad regex), from `rg
--replace` (prints the rewritten text to stdout but does not edit the
files in place — you still need a wrapper to apply), from `sd`
(in-place rewrite but no editor-driven preview / prune step), from
`comby` (structural / AST-aware patterns — different mental model, much
heavier syntax for the lexical case), and from IDE "Replace in Files"
panes (require the IDE, no good shell-pipeline story, often slow on
large monorepos). amber is the *batch text rename with an editor-buffer
prune step* shape: pick it specifically when you have a project-wide
rename whose regex matches both the things you want to change and a
handful of false positives, and you'd rather scroll through the
combined diff once and delete the bad hunks than build the perfect
exclusion regex. Do **not** pick it for AST-level refactors (use
`comby` or `ast-grep`), single-file scripted edits (use `sd` / `sed`),
or one-shot stdin transforms (use `rg --replace` piped to a file).

## Niche category
Editor-buffer preview-and-prune batch text replace — `rg`-class search
front + transactional in-place rewrite with a one-shot diff-buffer
review step.

## Repo
https://github.com/dalance/amber

## Version pinned
`v0.6.1` (latest tagged release, published 2026-02-24)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linux)
brew install amber

# Cargo (any platform)
cargo install amber

# Prebuilt binaries (Linux / macOS / Windows, x86_64 + aarch64)
# https://github.com/dalance/amber/releases/tag/v0.6.1
curl -L -o amber.zip \
  https://github.com/dalance/amber/releases/download/v0.6.1/amber-v0.6.1-x86_64-lnx.zip
unzip amber.zip && sudo mv amb{s,r} /usr/local/bin/
```

## Usage examples
```sh
# Plain recursive search (ripgrep-shaped front)
ambs 'fn old_name'

# Project-wide rename with editor-buffer prune step
# Opens $EDITOR with a combined diff of every hit; delete hunks you
# don't want; save + quit applies the surviving edits in one pass.
ambr 'fn old_name' 'fn new_name'

# Regex with backreference in the replacement
ambr 'log_(\w+)\(' 'logger.$1('

# Skip the editor preview (apply all matches blindly — use only
# when you've already proven the regex with `ambs` first)
ambr --no-interactive 'TODO\(alice\)' 'TODO(team)'

# Limit by file glob (still respects .gitignore by default)
ambs --include '*.rs' 'unsafe '

# Search inside hidden / ignored files too (escape hatch)
ambs --hidden --no-ignore 'AKIA[0-9A-Z]{16}'
```
