# teip

> **"Masking tape" for shell pipelines: apply a command to only
> selected parts of each line, leave the rest untouched** — a
> Rust CLI that lets you say "run `tr a-z A-Z` on just columns
> 2 and 4 of every line, but keep columns 1, 3, 5+ exactly as
> they are", or "base64-decode just the part of each line that
> matches this regex". Pinned to **v2.3.3**
> ([LICENSE](https://github.com/greymd/teip/blob/main/LICENSE),
> MIT).

Source: <https://github.com/greymd/teip>

## TL;DR

`teip` (pronounced "tape") solves a problem every shell user
hits eventually: "I want to apply a transformation to *part* of
each line, not the whole line, and stitch the result back
together". The classic workaround is `awk` (parse line, run
external on a sub-string via `getline cmd | ...`, re-emit), or
`cut` + `paste` + a temp file. `teip` collapses all of that
into one verb: select a region of each line (by `cut`-style
field range, by character range, by regex match, or by line
range), pipe *only those bytes* into the held-command, splice
the held-command's output back into the original positions,
emit. Because the masked-out parts never enter the held
command, you can run any tool — `tr`, `awk`, `sed`, `jq`, an
LLM CLI, even an interactive program — without it choking on
the surrounding context. The whole thing is one ~3 MB Rust
binary.

## Install

```bash
# Homebrew (macOS / Linux)
brew install greymd/tools/teip

# Cargo
cargo install teip --locked

# Pre-built deb / rpm
curl -LO https://github.com/greymd/teip/releases/download/v2.3.3/teip-2.3.3.x86_64.rpm
sudo rpm -i teip-2.3.3.x86_64.rpm
# or
curl -LO https://github.com/greymd/teip/releases/download/v2.3.3/teip_2.3.3-1_amd64.deb
sudo dpkg -i teip_2.3.3-1_amd64.deb

# Pre-built tarball (any Linux / macOS)
curl -LO https://github.com/greymd/teip/releases/download/v2.3.3/teip-2.3.3.x86_64-apple-darwin.tar.gz
tar xf teip-2.3.3.x86_64-apple-darwin.tar.gz
sudo install teip /usr/local/bin/

# verify
teip --version    # teip 2.3.3
```

Binary, no runtime, no config file, no state.

## Use it for

```bash
# Uppercase only the 2nd whitespace-separated field
echo "alice 42 dev" | teip -f 2 -- tr a-z A-Z
# → alice 42 DEV  (wait — that's field 3; -f 2 → 42 → 42)
echo "alice 42 dev" | teip -f 1 -- tr a-z A-Z
# → ALICE 42 dev

# Multiple field ranges
ls -l | teip -f 1,9- -- tr a-z A-Z
# → uppercase the perms column AND everything from the filename on,
#   leave size / date / owner / group untouched

# Custom delimiter (commas — like awk -F,)
echo "a,b,c,d,e" | teip -d , -f 2,4 -- tr a-z A-Z
# → a,B,c,D,e

# Character ranges (good for fixed-width files)
echo "2026-05-03 INFO  some message" | teip -c 1-10 -- date -d {} +%s --file=-
# → replace the date prefix with its unix timestamp

# Regex selection — only apply to bytes matching the pattern
cat access.log | teip -og '\d+\.\d+\.\d+\.\d+' -- nslookup {}
# → replace each IP in each line with its reverse-DNS, leaving
#   the surrounding log fields exactly intact

# Line-range selection (apply to a slice of stdin, not all of it)
cat file | teip -l 5,7,9-12 -- sed 's/foo/bar/'
# → only run sed on lines 5, 7, 9-12; pass others through unchanged

# Combine with jq for JSON-line transforms (apply jq only to a
# field, leave the rest of the line as a log prefix)
cat app.log | teip -og '\{.*\}' -- jq -c '.user_id'
# → log lines like `2026-05-03 12:00 {"user_id":"u123","x":1}`
#   become `2026-05-03 12:00 "u123"`

# Use in --solid mode: feed the held command the matched
# regions as one big stream, then splice back
cat file | teip -og '[A-Z]+' -s -- tr A-Z a-z
```

The held command runs **once** (not per line), receiving only
the masked-in bytes joined with newlines; its output is
re-spliced positionally. That makes it cheap even on huge
inputs and lets you compose with line-buffered tools.

## Why include it in a CLI catalog

1. **It fills a gap that has bugged shell users since the
   1970s.** Every long-time bash user has a half-remembered
   awk one-liner like `awk '{ "tr a-z A-Z" | getline x; ... }'`
   for "transform field 3, keep the rest". `teip` is the
   declarative version: select the region, hold any command,
   done. No sub-process plumbing, no `read -r`, no temp files.
   It is the missing UNIX verb: `cut` selects, `paste` joins,
   `teip` selects + transforms-in-place + joins.
2. **It composes with anything line-oriented.** Because the
   held command runs on a stream of bytes (the masked region of
   each line, joined by newlines) and emits the same shape,
   you can plug `sed`, `awk`, `jq`, `tr`, `nkf`, `iconv`,
   `base64`, `openssl`, even an HTTP client (`curl -s -d @-`)
   into the `--` slot. That makes it the universal
   per-region-transform without you having to write a Python
   script for each new pattern.
3. **Regex-region mode is the killer feature.** `teip -og
   '<pattern>' -- <cmd>` is the equivalent of "for each match
   in each line, replace it with `cmd <match>`" — exactly the
   shape of a thousand log-rewriting / IP-anonymising / token-
   stripping tasks. The rest of each line passes through
   byte-perfect, so the surrounding format (timestamps,
   delimiters, quoting) is preserved without you having to
   re-derive it.

For an LLM-CLI workflow, `teip -og '<some pattern>' -- llm -m
small 'rewrite this'` lets an agent send only the
should-be-rewritten spans to an LLM (cheap, fast, low context)
and let the surrounding template / log / code pass through
untouched — exactly the right tool for "redact PII in this log
stream", "translate just the comments in this source file", or
"summarise just the body of each email header block".

## Vs Already Cataloged

- **Vs `cut` (POSIX, not cataloged) and [`choose`](../choose/):**
  `cut` and `choose` *select and emit* a column / range; they
  drop everything else. `teip` *selects, transforms in place,
  and re-emits the whole line*. Where `cut -f2` gives you just
  field 2, `teip -f 2 -- tr a-z A-Z` gives you the original
  line with field 2 uppercased. They are not substitutes; they
  are the read vs. read-modify-write pair.
- **Vs [`sd`](../sd/) / `sed`:** `sd` and `sed` rewrite based
  on a pattern with a fixed replacement string (or a back-ref
  expression). `teip` rewrites based on a pattern with the
  output of an *arbitrary command* applied to the match. If
  your replacement is a literal string or a simple back-ref,
  use `sd` / `sed`. If your replacement is "run this match
  through `nslookup`" or "send each match to an LLM", you need
  `teip`.
- **Vs [`hck`](../hck/) / [`xsv`](../xsv/) / [`csvtk`](../csvtk/):**
  those are *table* tools (column-aware, header-aware, format-
  aware); `teip` is a *line* tool (delimiter-blind by default,
  works on logs and free text as well as CSV). For pure CSV /
  TSV with header semantics, the table tools are stronger; for
  free-form lines or files where you want to transform a
  regex-matched span, `teip` wins.
- **Vs [`srgn`](../srgn/):** `srgn` does language-aware (tree-
  sitter) selection + replace within source code; `teip` does
  regex / column / range selection + delegate-to-command on any
  byte stream. Different scope: `srgn` for "rename this Rust
  identifier in function bodies only", `teip` for "uppercase
  every IP in this log".
- **Vs `awk` with `getline cmd | ...`:** functionally similar
  but ~5× more verbose and easy to get wrong (subprocess per
  line, deadlocks if the held command buffers, awkward
  delimiter handling). `teip` is the same idea with a clean CLI
  surface and one subprocess for the whole pipeline.

## Caveats

- **The held command must produce the same number of records
  as it consumed.** `teip` splices output positionally — N
  matches in, N replacements out. A held command that drops
  lines (e.g. `grep -v`) or multiplies them desynchronises the
  splice and you get either truncated output or a mismatch
  error. Use `awk` / `sed` style tools that emit one line per
  input line.
- **Default field separator is whitespace, not tab.** Unlike
  `cut -f`, `teip -f` splits on runs of whitespace by default.
  Pass `-d $'\t'` for strict tabs.
- **Regex mode is line-by-line by default.** `teip -og` matches
  per line; for cross-line patterns use `-G` (oniguruma) or
  pre-flatten with `tr '\n' ' '`.
- **Solid mode (`-s`) changes semantics.** `-s` joins all
  matches into one stream and feeds them as a single payload
  to the held command, then splits the output by newlines and
  re-splices. Powerful (lets the held command see all matches
  at once — good for batched LLM calls), but a held command
  that emits a different number of `\n`-separated chunks than
  it received will scramble the splice.
- **No streaming for huge inputs in `-s` mode.** Solid mode
  buffers all matches before invoking the held command; for a
  multi-GB log this means RAM proportional to total match
  bytes. Per-line mode (`-og` without `-s`) is fully streaming.
- **Not POSIX; not on every box.** Unlike `cut` / `awk` /
  `sed`, `teip` is not preinstalled anywhere. Scripts you
  share need to either bundle the binary, fall back to an awk
  approximation, or assume the recipient runs `cargo install
  teip`.
