# loc

> **A Rust CLI that counts lines of code across a tree, broken
> down by language, in a `cloc`-shaped table — but parallel and
> typically much faster than the Perl `cloc` it imitates. One
> ~2 MB static binary, no runtime, no config.** Pinned to
> **v0.4.1**,
> [LICENSE-MIT](https://github.com/cgag/loc/blob/master/LICENSE-MIT)
> / [LICENSE-APACHE](https://github.com/cgag/loc/blob/master/LICENSE-APACHE),
> dual MIT / Apache-2.0.

Source: <https://github.com/cgag/loc>

## TL;DR

`loc` walks a directory tree, classifies each file by extension,
counts code / comment / blank lines per language, and prints a
sorted table. Same column layout as `cloc` — `Language | Files |
Lines | Blank | Comment | Code` — but written in Rust with
parallel I/O, so a tree that takes `cloc` 30 seconds to scan
finishes in 2-3 with `loc`. The whole tool is one ~2 MB binary
with `--exclude` and `--unrestricted` flags; everything else is
the table. `loc` is *not* trying to match `cloc`'s exhaustive
language-detection edge cases (shebang sniffing, bracketed
comment forms in obscure languages) — it covers the ~150
mainstream languages well and runs fast.

## Install

```bash
# Homebrew (macOS / Linux)
brew install loc        # currently 0.4.1

# Cargo (any platform with a Rust toolchain)
cargo install loc

# Pre-built binaries
# https://github.com/cgag/loc/releases

# Verify
loc --version           # loc 0.4.1
```

## License

Dual-licensed MIT / Apache-2.0 — see
[LICENSE-MIT](https://github.com/cgag/loc/blob/master/LICENSE-MIT)
and
[LICENSE-APACHE](https://github.com/cgag/loc/blob/master/LICENSE-APACHE).
Pick whichever you prefer; both are permissive and bundle-friendly.

## One Concrete Example

```bash
# 1. count everything in $PWD
loc

# Output:
# --------------------------------------------------------------------------------
#  Language             Files        Lines        Blank      Comment         Code
# --------------------------------------------------------------------------------
#  Rust                    42        12834         1502         1037        10295
#  Markdown                17         3401          612            0         2789
#  TOML                    11          348           42           19          287
# --------------------------------------------------------------------------------
#  Total                   70        16583         2156         1056        13371
# --------------------------------------------------------------------------------

# 2. count only Rust + Python, exclude vendor / test fixtures
loc --include rs,py --exclude vendor --exclude testdata

# 3. include hidden files and gitignored paths (default skips .git, target/, etc.)
loc --unrestricted

# 4. machine-readable for CI gating (line count delta over baseline)
loc --files | awk '{sum += $5} END {print sum}'
```

## Niche It Fills

**The fast Rust answer to `cloc`.** Same table, same workflow,
~10-20× faster on large trees because it parallelises across
files and uses Rust's `walkdir` + `ignore` (the `ripgrep` walker)
to honour `.gitignore` by default. Pick `loc` for routine
"how big is this codebase?" runs in CI or on the command line
when the language coverage of `cloc` is overkill.

## Why use it

Three things `loc` does that pay off immediately:

1. **Parallel by default.** Scans across all cores; a 1 GB
   monorepo finishes in a few seconds instead of a minute.
2. **Honours `.gitignore`.** Uses the same walker as `ripgrep`,
   so `target/`, `node_modules/`, `dist/`, `.venv/` are skipped
   automatically — no `--exclude-dir` chain to maintain.
3. **One binary, zero config.** No language definition file to
   edit, no Perl runtime, no JSON output schema to learn —
   table goes to stdout, you grep / awk / wc from there.

For an LLM-CLI workflow, `loc` is the cheap "is this repo small
enough to pack into context?" probe — pipe `loc` into a
threshold check before invoking
[`repomix`](../repomix/) /
[`code2prompt`](../code2prompt/) /
[`files-to-prompt`](../files-to-prompt/), so an agent does not
try to pack a 4-million-LOC tree into a 200 k context window.

## Vs Already Cataloged

- **Vs [`tokei`](../tokei/):** `tokei` is the more featured
  modern Rust line-counter — same speed class as `loc`, plus
  per-file output, JSON / YAML / CBOR / OpenMetrics export, a
  `--sort` flag, and a regularly-maintained language definition
  file (~270 languages today vs `loc`'s ~150). Pick `tokei` for
  any non-trivial use; `loc` survives in this catalog as the
  smaller, simpler, "just give me the table" alternative.
- **Vs [`scc`](../scc/):** `scc` is the third Rust-class
  contender — adds COCOMO cost-estimation and complexity
  scoring on top of LOC, written in Go. Pick `scc` when you
  want the cost / complexity columns, `loc` when you want the
  minimal table.
- **Vs [`cloc`](../cloc/):** `cloc` is the Perl original with
  the broadest language coverage and the most mature edge-case
  handling (heredocs, embedded language detection). Pick
  `cloc` when correctness on obscure languages matters more
  than speed; `loc` when you want the same answer in a tenth
  of the wall-clock time.
- **Vs [`tokei`](../tokei/) + [`scc`](../scc/) (the trio):**
  the three are largely interchangeable for the common case.
  `tokei` has won the most users; `loc` and `scc` survive on
  specific differentiators (`scc` for cost estimation, `loc`
  for "smallest binary that does the job").

## Caveats

- **Smaller language coverage than `tokei` / `cloc`.** ~150
  vs ~270 languages. Mainstream languages are covered; obscure
  / templating / config-as-code languages may be miscounted or
  classified as "Unrecognized."
- **No structured output.** Stdout table only — no JSON / YAML
  flag for clean CI parsing. You scrape the table or pipe
  through `awk`. `tokei -o json` is the right choice if you
  need machine-readable output natively.
- **Maintenance is quiet.** Last point release is some years
  old; the tool works because the LOC-counting problem is
  largely solved, but new languages do not land. For active
  maintenance pick `tokei`.
- **No per-file breakdown by default.** `--files` lists files
  with counts, but the per-language `--by-file` cross-tab
  modes that `cloc` and `tokei` offer are absent.
- **Default skips hidden + gitignored.** Surprising the first
  time — pass `--unrestricted` to count everything (including
  `.git/` itself, which you almost never want).
