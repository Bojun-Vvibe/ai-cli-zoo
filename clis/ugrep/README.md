# ugrep

- **Repo:** https://github.com/Genivia/ugrep
- **Version:** 7.8.0 (tagged 2026-04-29; current stable on the 7.x line)
- **License:** BSD-3-Clause — see [`LICENSE.txt`](https://github.com/Genivia/ugrep/blob/master/LICENSE.txt)
- **Language:** C++ (RE/flex-based DFA matcher; SIMD-accelerated literal scan via `libsimdpp` / hand-tuned AVX2 + NEON; mmap I/O; pthread worker pool)
- **Install:** `brew install ugrep` · `apt install ugrep` · `dnf install ugrep` · `pacman -S ugrep` · or build from source (`./build.sh && sudo make install`); ships two binaries, `ugrep` (the search engine) and `ug` (an interactive TUI variant with persistent config + ncurses pager)

## Overview

`ugrep` is a `grep`-compatible search tool that the
author wrote because `grep` / `egrep` / `fgrep` / `pcre2grep`
each cover slightly different regex dialects, none of
them index archives or PDFs, and `rg` (ripgrep) trades
a few corners of POSIX BRE/ERE for speed. ugrep keeps
**every** grep dialect (`-G` BRE, `-E` ERE, `-P`
PCRE2, plus `-Z` fuzzy / approximate matching with
configurable edit distance, plus `-X` hexadecimal,
plus `--query` interactive TUI mode), recursively
descends with `.gitignore` honored (`--ignore-files`),
and **searches inside archives, compressed files, PDFs,
and Office docs** without unpacking them: `ugrep -z
"TODO" .` matches inside `.gz` / `.bz2` / `.xz` /
`.lzma` / `.lz4` / `.zstd` / `.br` / `.zip` / `.7z` /
`.tar` / `.cpio` / `.pax` archives, plus `--filter
"pdf:pdftotext,doc:antiword,docx:docx2txt"` pipes
through user-defined extractors. Output speaks every
common shape: classic `file:line:match`,
`--json` / `--xml` / `--csv` for downstream tools,
`-Iz` for indexed binary scan, `--colors=auto` with
fine-grained per-element colour control, and a
`--replace` mode that emits sed-style rewrites
(without the file-locking pitfalls of in-place sed).
The interactive `ug --query` mode opens a real TUI
with live regex preview, hit-list navigation, file
preview, and a `CTRL-Y` viewer hand-off — the closest
thing to a "fzf for regex search of a tree" in the
catalog.

## Niche

**The "grep that does everything" — every grep dialect
including PCRE2 and approximate matching, recursive
search of compressed archives and PDF / Office docs
without unpacking, and a real TUI mode — in one
single-binary BSD-3 C++ tool.** The role is "the
search tool you reach for when `rg` cannot reach into
the `.tar.zst` your CI dropped, when `grep -P` fails
because the regex needs PCRE recursion, when you need
fuzzy match because the source has typos, or when you
want to interactively narrow a regex against a 1 GB
log dir before piping into `awk`." The competing
universe is `ripgrep` / `the_silver_searcher` (`ag`) /
`grep` / `pcre2grep` / `git grep` — see comparisons
below.

## When to use

- You need to search inside archives without
  unpacking: `ugrep -z "panic" build-artifacts/` walks
  `.tar.zst` / `.zip` / `.7z` / nested archives in
  one pass.
- You need to grep **inside PDF / DOCX / XLSX**:
  `ugrep --filter='pdf:pdftotext -layout - -,docx:pandoc -t plain' "ATC clearance" docs/`.
- You need **PCRE2 with recursion / lookbehind / named
  captures** that `grep -P` partially supports and `rg`
  exposes only as `--pcre2`: `ugrep -P
  '(?<=Authorization:\s)Bearer\s+\K[A-Za-z0-9._-]+' logs/`
  extracts the token without surrounding context.
- You need **approximate / fuzzy matching** for
  OCR-noisy or hand-typed input:
  `ugrep -Z2 "kuberntes" docs/` matches `kubernetes`
  / `kuberentes` / `kuberntes` within edit distance 2.
- You want to **interactively narrow a regex** against
  a tree before committing to a pipeline:
  `ug --query` opens a TUI where every keystroke
  re-runs the regex live with file previews and hit
  navigation.
- You need **structured output** for downstream tools:
  `ugrep --json -P 'pat' .` emits one JSON object per
  match with `path`, `line`, `column`, `match`, named
  capture groups, and surrounding context.
- You need **hex / binary scan**: `ugrep -X
  '\xde\xad\xbe\xef' /var/lib/foo` scans for byte
  patterns with byte-offset output.
- You want **a single binary that subsumes `grep` /
  `egrep` / `fgrep` / `pcre2grep` / `agrep` / `tre`**
  with the same flags those tools used.

## When NOT to use

- You want **the absolute fastest single-tree
  recursive grep** with the smallest dependency
  surface — `rg` (ripgrep) is typically faster on the
  pure "match a literal in a million source files"
  case because it spends less time on regex
  generality, and it has zero runtime deps. Pick `rg`
  for the hot loop, `ugrep` for the corner cases `rg`
  cannot reach.
- You only ever need POSIX BRE and `--include` —
  plain `grep` is everywhere, has zero install cost,
  and is well-understood by every shell user.
- You need **AST-level structural search**, not
  regex — pick [`ast-grep`](../ast-grep/) /
  [`comby`](../comby/) / [`srgn`](../srgn/) /
  [`grep-ast`](../grep-ast/) instead. Regex (no
  matter how powerful) cannot tell function bodies
  from string literals; an AST tool can.
- You need **file-aware semantic search** with
  embeddings — pick [`seagoat`](../seagoat/) or
  [`khoj`](../khoj/) or [`grep-ast`](../grep-ast/);
  ugrep is purely lexical.

## Comparison vs alternatives in zoo

- [`ripgrep`](../ripgrep/) — Rust, single binary, the
  performance-leader baseline for recursive
  source-tree grep. Honors `.gitignore` by default,
  has `--type` aliases for every common language,
  ships in just about every distro now. Pick `rg`
  when the workload is "regex over a source tree" and
  the regex is plain ERE / PCRE without recursion;
  pick `ugrep` when you need archive descent, PDF /
  Office filtering, fuzzy matching, or the
  interactive `ug --query` TUI.
- [`ag`](../ag/) — The Silver Searcher (C, single
  binary). Older, slower than rg / ugrep, still
  ships in many distros; effectively superseded by
  `rg` for most users.
- [`ast-grep`](../ast-grep/) — Tree-sitter-powered
  structural search. Orthogonal — pick `ast-grep` for
  "find all calls to `foo(x, y)` regardless of
  formatting"; pick `ugrep` for textual / regex
  search across archives, PDFs, and binary files.
- [`comby`](../comby/) — language-aware structural
  search-and-replace, like `ast-grep` but with a
  template-matching syntax. Same orthogonality as
  above.
- [`srgn`](../srgn/) — surgical regex + tree-sitter
  hybrid. Closer to `ast-grep` in spirit; orthogonal
  to ugrep's archive / PDF / fuzzy story.
- [`grep-ast`](../grep-ast/) — `rg` wrapper that
  expands matches to enclosing AST nodes (function /
  class / method bodies). Pure complement on top of
  `rg`; ugrep does not have an equivalent wrapper at
  this snapshot.
- [`fselect`](../fselect/) / [`fd`](../fd/) — file
  *finders*, not content searchers. Complementary —
  `fd '\.tar\.zst$' . | xargs ugrep -z 'pat'` is a
  reasonable composition.
- [`htmlq`](../htmlq/) / [`xq`](../xq/) /
  [`gojq`](../gojq/) / [`fq`](../fq/) — structured
  query for specific formats. ugrep is the
  format-agnostic regex layer; reach for the
  structured tool when the file shape is known and
  stable.

## Why it earns a slot in an AI-native workflow

LLM context-gathering is increasingly "find every
mention of `X` across a heterogeneous archive of
sources": the source tree, plus the
`docs/*.pdf` vendor manuals, plus the `transcripts/
*.docx` meeting notes, plus the
`build-artifacts/*.tar.zst` CI bundles, plus the
`logs/*.gz` rotated logs. `rg` covers the source
tree well; ugrep covers everything else in the same
flag-compatible way (`ugrep -rz --include='*.tar.zst'
--include='*.pdf' --include='*.docx' --include='*.gz'
'<query>' .` is a one-liner an agent can emit). The
`--json` mode means an agent can consume the matches
as structured JSON without parsing line-formatted
output, and `ug --query` is a TUI a human can use to
*verify* an agent's regex against the corpus before
shipping the resulting context into a prompt. Fuzzy
mode (`-Z`) is also notably useful for grepping LLM
outputs themselves, where typos and near-matches in
generated identifiers are common.

## Example invocations

```bash
# Drop-in grep replacement
ugrep -rn "TODO" src/

# PCRE2 with named captures and lookbehind
ugrep -P -rn '(?<=Authorization:\s)Bearer\s+\K[A-Za-z0-9._-]+' logs/

# Recursive search inside compressed archives — no unpacking
ugrep -rz "panic" build-artifacts/   # descends .gz .xz .zst .zip .7z .tar

# Search inside PDFs by piping through pdftotext
ugrep --filter='pdf:pdftotext -layout - -' -rn 'ATC clearance' docs/

# Search inside Office docs (DOCX / XLSX) via pandoc
ugrep --filter='docx:pandoc -t plain,xlsx:in2csv' -rn 'invoice' notes/

# Approximate / fuzzy match (edit distance ≤2)
ugrep -Z2 "kuberntes" docs/

# Hex / binary scan
ugrep -X '\xde\xad\xbe\xef' /var/lib/foo

# JSON output for downstream tools (agents, scripts)
ugrep --json -P 'class\s+(?<name>\w+)' src/

# Replace mode (sed-style rewrite, prints to stdout)
ugrep -P --replace='Bearer ***' 'Bearer\s+[A-Za-z0-9._-]+' app.log

# Interactive TUI: live regex preview, hit navigation, file preview
ug --query
# ... type a regex, see live matches, ENTER to open in $EDITOR

# Combine with fd for "files matching X containing Y"
fd '\.tar\.zst$' . | xargs ugrep -lz 'panic'
```
