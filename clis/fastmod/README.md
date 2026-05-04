# fastmod

> **Large-scale interactive code refactoring** — a Rust rewrite of
> Facebook's `codemod` that walks a tree, finds regex matches in
> parallel across all files, and shows each hunk with a y/n/e/A
> prompt so you can approve, reject, or edit replacements before
> they hit disk. Pinned to **v0.4.4** (released 2024,
> [LICENSE](https://github.com/facebookincubator/fastmod/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/facebookincubator/fastmod>

## TL;DR

`fastmod <regex> <replacement> <path…>` does what `sed -i` wishes
it could: it finds every match in parallel using ripgrep-grade
walking, and for each hunk shows you a colored diff with a
prompt — `y` accept, `n` skip, `e` open in `$EDITOR`, `A` accept
all remaining in this file, `q` quit. Replacement strings use
familiar `$1` capture groups. Filtering by extension
(`-e py,pyi`), by glob (`-g 'src/**'`), and by file regex is
built in. Because it shows context and asks per-hunk, it is the
right tool for the "rename a thing across 1,200 files but the
match has 6 false positives I have to dodge" job — exactly the
shape of refactor that makes pure `sed -i` dangerous.

It is not a syntax-aware refactoring tool (use
[`comby`](https://comby.dev), [`ast-grep`](https://ast-grep.github.io),
or [`srgn`](https://github.com/alexpovel/srgn) for that). It is
the better `sed` for codebases.

## Why it's interesting

Most "rewrite across a repo" tooling forces a binary choice:
fully automatic (`sed -i`, `find … -exec`) which is dangerous,
or fully manual (open each file, find/replace) which does not
scale. `fastmod` is the missing middle: parallel search, regex
power, but human-in-the-loop per hunk with a one-keystroke
diff review. That UX was originally invented at Facebook
(Python `codemod` from 2007) and then rewritten in Rust for
speed; `fastmod` is the canonical descendant. It is the tool
you reach for during library upgrades, deprecation sweeps, and
"the API renamed `getX` to `x` but only in some contexts"
migrations.

## Install

```bash
# Cargo (cross-platform)
cargo install fastmod

# macOS
brew install fastmod

# Arch
sudo pacman -S fastmod    # community

# verify
fastmod --version    # fastmod 0.4.4
```

## Examples

```bash
# rename an identifier across a Python project, asking per match
fastmod 'getUserName' 'get_user_name' -e py

# regex with capture groups: rewrite `assertEqual(x, y)` →
# `assertEqual(y, x)` (swap arg order) — interactively
fastmod 'assertEqual\((\w+),\s*(\w+)\)' 'assertEqual($2, $1)' -e py

# limit to a subdirectory and dodge tests
fastmod -d src --ignore-dir tests 'OldClient' 'NewClient' -e ts,tsx

# accept everything without prompting (use only after a dry-run)
fastmod --accept-all 'http://internal' 'https://internal' -e yaml,json

# multi-line regex (e.g. rewrite a 2-line decorator pattern)
fastmod -m '@deprecated\n\s*def (\w+)' 'def $1' -e py

# dry-run by piping `n` to every prompt
yes n | fastmod 'TODO' 'FIXME' -e rs

# filter by file content first (only files matching pattern X
# get considered for replacement Y)
fastmod --include-file-pattern 'def test_' 'assert ' 'self.assertTrue(' -e py
```

## Use when

- You are doing a regex-shaped refactor across hundreds or
  thousands of files and the match has enough false positives
  that you do *not* trust `sed -i`.
- You are sweeping a deprecation: rename function, swap
  argument order, change import path. Each hit needs a
  half-second human glance, not a full `git diff` review at
  the end.
- You want `git grep -E pattern | xargs sed` ergonomics but
  with a per-hunk diff and accept/reject prompt.
- You want a faster, parallel, ripgrep-walking replacement
  for the original Python `codemod`.

Skip `fastmod` for syntax-aware refactors that need to
respect parsing (renaming a method without hitting strings,
comments, or unrelated identifiers) — reach for
[`comby`](https://comby.dev),
[`ast-grep`](https://ast-grep.github.io), or
[`srgn`](https://github.com/alexpovel/srgn) instead.
For language-server-grade renames, use the LSP `rename` action
in your editor.
