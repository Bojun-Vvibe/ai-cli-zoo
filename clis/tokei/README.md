# tokei

> **`cloc` rewritten in Rust — fast, parallel, accurate code-line
> counter.** A single Rust binary that walks a directory tree and
> reports per-language line counts (code / comment / blank) for
> ~250 languages, parsed with real per-language tokenizers (so
> `/* */` inside a string literal is not counted as a comment),
> respects `.gitignore` by default, and finishes a 1-million-LOC
> repo in well under a second on a laptop. Pinned to **v14.0.0**
> (dual MIT / Apache-2.0, see
> [LICENCE-MIT](https://github.com/XAMPPRocky/tokei/blob/master/LICENCE-MIT)
> and
> [LICENCE-APACHE](https://github.com/XAMPPRocky/tokei/blob/master/LICENCE-APACHE)).

Source: <https://github.com/XAMPPRocky/tokei>

## TL;DR

`tokei` answers "how big is this codebase, by language" in one
command, with numbers you can trust. The classic answer was
`cloc` (Perl), which is correct but slow on large repos and
hand-bombs Rust async syntax / TSX / Vue SFCs / modern Nix /
recent Swift. `tokei` parses each file with a per-language
state machine (string-aware, nested-comment-aware) so the
"code vs comment vs blank" split is right on the tricky cases
(Rust raw strings, Python triple-quoted docstrings,
`/* /* nested */ */` in Rust / D / Swift, Vue / Svelte / MDX
multi-language files). It walks the tree in parallel via
`rayon`, respects `.gitignore` (and supports `--no-ignore` to
override), and outputs human-readable tables, JSON, YAML, CBOR,
or an OpenMetrics format that drops straight into Prometheus.

## Install

```bash
# Homebrew
brew install tokei

# Cargo
cargo install tokei

# Linux package managers
# Arch: pacman -S tokei
# Nix: nix-env -iA nixpkgs.tokei
# Fedora: dnf install tokei

# Windows
# scoop install tokei
# winget install XAMPPRocky.tokei

# from a GitHub release
curl -Lo tokei.tar.gz "https://github.com/XAMPPRocky/tokei/releases/download/v14.0.0/tokei-aarch64-apple-darwin.tar.gz"
tar xf tokei.tar.gz
sudo install tokei /usr/local/bin/

# verify
tokei --version    # tokei 14.0.0
```

Zero config — `tokei` in any directory just works. To exclude
vendored / generated files beyond `.gitignore`, drop a
`.tokeignore` file at the repo root with extra patterns.

## License

Dual MIT / Apache-2.0 — see
[LICENCE-MIT](https://github.com/XAMPPRocky/tokei/blob/master/LICENCE-MIT)
and
[LICENCE-APACHE](https://github.com/XAMPPRocky/tokei/blob/master/LICENCE-APACHE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. summarise the current repo
tokei
# ===============================================================
#  Language            Files        Lines         Code     Comments     Blanks
# ===============================================================
#  Rust                  142        38420        29104         3211       6105
#  TypeScript             87        12056         9874          412       1770
#  Markdown               24         3210            0         3210          0
# ---------------------------------------------------------------
#  Total                 253        53686        38978         6833       7875
# ===============================================================

# 2. machine-readable output for CI dashboards
tokei --output json > codebase-size.json
tokei --output openmetrics | curl -X POST --data-binary @- \
    http://prom-pushgateway/metrics/job/repo-size

# 3. compare two snapshots (size growth this PR)
git stash && tokei -o json > /tmp/base.json && git stash pop
tokei -o json > /tmp/head.json
jq -s '.[1].Total.code - .[0].Total.code' /tmp/base.json /tmp/head.json
# → 412   (this PR added 412 lines of code, net of comments / blanks)

# 4. only specific languages
tokei -t Rust,TypeScript

# 5. include vendored / gitignored dirs you would normally skip
tokei --no-ignore vendor/

# 6. per-file breakdown (find the bloated outliers)
tokei --files | sort -k4 -n -r | head -20

# 7. sort by comment ratio (find under-documented modules)
tokei --files -o json | jq -r '
  .Rust.reports[] | "\(.stats.comments)/\(.stats.code)\t\(.name)"
' | awk -F/ '$1/$2 < 0.05 {print}'
```

## Niche It Fills

**The codebase-size primitive.** Onboarding docs, technical
blog posts, internal capacity planning, "is this monorepo
getting out of hand" PR-time gates — all want a fast, accurate,
machine-parseable LOC count. `tokei` is small enough to drop
into pre-commit / CI and fast enough that nobody complains.

## Why use it

Three things `tokei` does that `cloc` does not, that pay back
the switching cost:

1. **Sub-second on million-LOC repos.** `rayon`-parallel tree
   walk + tight Rust state machines mean `tokei
   linux/` (~30M LOC) finishes in seconds, not minutes. Cheap
   enough to run on every PR.
2. **Modern-syntax accuracy.** Rust raw strings (`r#"..."#`),
   Swift multi-line string literals, Vue / Svelte / MDX SFCs
   (multiple language sections in one file, each counted
   correctly), TSX, Nix, Zig, Gleam — `cloc`'s regex-based
   parser miscounts these; `tokei`'s per-language state machine
   gets them right. The output's `Code` column is the number a
   reviewer would actually agree with.
3. **First-class machine output.** `--output json/yaml/cbor`
   plus an `openmetrics` exporter make "track repo size over
   time on a Prometheus dashboard" a one-line cron, and
   `--files` per-file mode plus `jq` is enough for "which
   modules are over X lines" custom queries — no separate
   reporter tool needed.

For an LLM-CLI workflow, `tokei -t Rust --files -o json` is the
cheapest way to feed an agent the shape of a codebase before
asking it to plan refactors — total LOC per file, comment
density, language mix. The JSON output drops straight into a
context packer ([`files-to-prompt`](../files-to-prompt/),
[`code2prompt`](../code2prompt/), [`repomix`](../repomix/)) as
a budget hint.

## Vs Already Cataloged

- **Vs [`tokei`'s upstream `cloc`](https://github.com/AlDanial/cloc)
  (not cataloged):** same job, `tokei` is one to two orders of
  magnitude faster on big repos, more accurate on modern syntax,
  ships as a static binary instead of a Perl script. `cloc` is
  still the right answer if you need its `--diff` mode (LOC
  added / removed between two trees) or the breakdown by
  third-party identifier — `tokei` does not match those exactly.
- **Vs [`scc`](https://github.com/boyter/scc) (not cataloged):**
  closest peer (Go, fast, similar feature set, adds COCOMO cost
  estimates and a "complexity" heuristic). Pick `scc` when you
  want the cost / complexity columns; pick `tokei` when you want
  the broadest language coverage and the cleanest JSON / OpenMetrics
  exporter.
- **Vs [`hyperfine`](../hyperfine/) / [`bottom`](../bottom/) /
  [`bat`](../bat/):** orthogonal — those measure runtime
  behaviour or render text; `tokei` measures source size. Often
  used together in a "repo health dashboard" recipe.

## Caveats

- **Counts source lines, not commits / authors / churn.** For
  contributor- or churn-shaped questions reach for
  [`git-quick-stats`](https://github.com/arzzen/git-quick-stats)
  or `git log --shortstat` post-processing. `tokei` is a static
  snapshot.
- **Generated / vendored files inflate counts unless
  excluded.** `tokei` respects `.gitignore` by default but a
  generated `_pb2.py` / `*.gen.go` / `dist/` checked into git
  still counts. Add a `.tokeignore` with the relevant patterns
  for honest numbers; double-check with `tokei --files | sort
  -k4 -nr | head` to see what is dominating.
- **Markdown / JSON / TOML report 0 code lines.** By design —
  data files have no `code` line concept; the column is `0`
  and the lines land in `comments` (Markdown) or `blanks` /
  total only. If you want them in the `Code` column for a
  vibes-based "total lines" headline, sum
  `code+comments+blanks` from the JSON.
- **No `--diff` mode.** `cloc` can compare two trees and report
  net added / removed; `tokei` cannot. The `jq`-based two-snapshot
  recipe in the example above is the workaround.
- **Language detection is extension-first.** Files without an
  extension or with an unusual one (`.tmpl`, `.in`, `.mustache`)
  may end up in the `(unknown)` bucket; map them via
  `--languages` config or rename.
