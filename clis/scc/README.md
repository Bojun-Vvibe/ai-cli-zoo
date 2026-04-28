# scc

- **Repo:** https://github.com/boyter/scc
- **Version:** v3.5.0 (latest stable, 2025)
- **License:** MIT / Unlicense dual ([LICENSE](https://github.com/boyter/scc/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install scc` · `go install github.com/boyter/scc/v3@latest` · `apt install scc` (Debian/Ubuntu trixie+) · binary releases on the GitHub release page

## What it does

`scc` (Sloc, Cloc and Code) is a fast multi-language source-code counter and
COCOMO estimator. You point it at a directory; it walks the tree in parallel,
auto-detects each file's language by extension + shebang + content sniff,
counts physical lines, blank lines, comment lines, and effective code lines,
and prints a per-language table sorted by code lines. It supports 250+
languages out of the box including the long tail (Coq, Crystal, Carbon, Zig,
Mojo, Roc, Gleam, Pkl, Cue, HCL, Bicep, Nushell), recognises generated files
and vendor directories via `.gitignore` / `.ignore` plus its own ignore
heuristics, and respects the same path filters as `rg` so per-repo conventions
just work. The thing that distinguishes it from `cloc` and `tokei` is twofold:
parallel I/O with a custom file walker that saturates an SSD instead of a
single core, and an in-tree complexity score per file (cyclomatic-style:
counts branching keywords per language) so the table doubles as a "where is
the gnarly code" map. Output formats include the default human table, CSV,
JSON, HTML for embedding, OpenMetrics for Prometheus scrape, SQL inserts, and
a per-file mode (`--by-file`) that turns it into "biggest / most complex
files" radar. The COCOMO estimate (`--avg-wage 75000`) is more conversation
piece than budget tool, but the per-language SLOC + complexity together are
genuinely useful for sizing a refactor.

## When to pick it / when not to

Pick `scc` whenever you would have reached for `cloc` or `tokei`. It is the
fastest of the three on a cold cache (parallel walker + zero-alloc parser),
the only one that ships a per-file complexity score in the same pass, and
the only one whose output formats include something a Prometheus exporter
can scrape directly. Reach for it on day one of a new codebase ("what
language is this even in"), before kicking off a refactor ("which 20 files
hold half the complexity"), in CI as a soft size budget (fail if the
`vendored:` row crosses N% of total), and for one-shot reports to a
non-engineer ("here is the HTML table, sorted by lines"). It pairs with
[`tokei`](../tokei/) (cross-check counts when something looks off — the two
classifiers disagree on edge cases), with [`erdtree`](../erdtree/) (tree+du
for the same directory you just counted), and with
[`hyperfine`](../hyperfine/) when you want to time the count itself across
codebases.

Skip it when you need **AST-level metrics** — function count, parameter
count, fan-in / fan-out, halstead. `scc`'s complexity score is keyword
counting, not real cyclomatic; for that go to `lizard`, `radon` (Python), or
language-specific tools. Skip it when **classification accuracy on
ambiguous extensions** matters more than speed (`.h` is C? C++? Objective-C?
— `cloc` has a more conservative heuristic, `tokei` has a more aggressive
one, scc is in the middle). Skip it for **license detection**, **dependency
graphing**, or anything beyond counting; `scc` does not know what your code
imports. And skip it on **submodules you have not initialised** — it counts
what is on disk; an empty submodule directory contributes zero lines and
will silently understate the codebase.

## Example invocations

```bash
# Count the current repo, default human table
scc

# Count a specific path, sorted by code lines (default)
scc src/

# Per-file mode: biggest / most complex files
scc --by-file --sort complexity src/ | head -30

# CI gate: fail if any single file is over 1000 effective lines
scc --by-file --format json src/ \
  | jq -e 'all(.[]; .Code <= 1000)'

# JSON for downstream tooling
scc --format json . > sloc.json

# CSV for a spreadsheet
scc --format csv . > sloc.csv

# HTML report to drop into a wiki or PR description
scc --format html-table . > sloc.html

# OpenMetrics for Prometheus scrape (paired with a static file exporter)
scc --format openmetrics . > /var/lib/node_exporter/scc.prom

# Restrict to a language subset
scc --include-ext go,rs,py .

# Ignore vendored / generated paths beyond the defaults
scc --exclude-dir vendor,third_party,generated .

# Estimate effort with a custom average wage
scc --avg-wage 90000 .
```
