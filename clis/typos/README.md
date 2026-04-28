# typos

- **Repo:** https://github.com/crate-ci/typos
- **Version:** v1.45.2 (latest stable, 2026)
- **License:** MIT or Apache-2.0 ([LICENSE-MIT](https://github.com/crate-ci/typos/blob/master/LICENSE-MIT), [LICENSE-APACHE](https://github.com/crate-ci/typos/blob/master/LICENSE-APACHE))
- **Language:** Rust
- **Install:** `brew install typos-cli` · `cargo install typos-cli` · `pacman -S typos` · prebuilt binaries on the GitHub release page · binary name is `typos`

## What it does

`typos` is a source-code spell checker built around a hand-curated
dictionary of **real-world misspellings of code identifiers**, not
of English prose. It scans your repo for things like `recieve`,
`occured`, `seperator`, `lenght`, `accomodate`, `existant`,
`paramter`, and the thousand other near-miss identifiers that
slip past every other spell checker because they tokenize on
`camelCase`, `snake_case`, and `kebab-case` and ignore code
keywords. The dictionary is conservative: only corrections with
no plausible alternate meaning are flagged, so false-positive rate
is extremely low — low enough that `typos` is safe to run as a
required CI check, with `--write-changes` allowed to auto-fix.

It reads `.typos.toml` for per-repo overrides (allow specific
"misspellings" that are actually project nouns, scope checks to
certain extensions, exclude generated dirs). It respects
`.gitignore` by default. Speed is in the same ballpark as
`ripgrep`: a 100k-LOC repo scans in well under a second on a
laptop.

## When to pick it / when not to

Reach for `typos` when you want a **zero-config spell pass on
identifiers, comments, strings, and docs** that you can wire into
pre-commit and CI without flooding contributors with noise. It is
the right tool when your repo has many contributors, when
documentation drift is a real problem, and when you have already
been bitten by a typo in a public API name that you cannot rename
without a deprecation cycle. It is also excellent as an
auto-fixer (`typos --write-changes`) on a stale repo: one
commit, a hundred small wins.

Skip it for prose-heavy repos where you actually want a full
English dictionary (use [`vale`](https://vale.sh) or
[`languagetool`](https://languagetool.org) instead). Skip it
when your project legitimately uses a lot of made-up terms
(crypto, biology, internal product names) without maintaining a
`.typos.toml` allowlist — you'll spend more time tuning than
fixing. It also will not catch grammatical errors, missing words,
or wrong-but-spelled-correctly usages.

## Why it matters in an AI-native workflow

LLM-generated code and docs ship a steady stream of subtle
identifier typos: `intialize`, `arguements`, `responce`,
`succesfully`. They often slip through human review because the
surrounding code looks correct and the eye glides over a one-letter
slip. Once shipped, a typo in a public symbol becomes a permanent
API tax — every caller must repeat the misspelling, every grep
must match both forms. Running `typos` as a pre-merge check
intercepts these before they become load-bearing. In agent loops,
adding `typos` to the post-edit lint pass gives the agent a
deterministic signal to self-correct, much cheaper than asking
another LLM to proofread.

## Example invocations

```bash
# Scan the current repo, exit non-zero on findings
typos

# Auto-fix everything the dictionary considers unambiguous
typos --write-changes

# Scope to a single path (e.g. only the docs)
typos docs/

# Show what would change without writing
typos --diff

# Limit to a single file extension via config or CLI
typos --type rust

# Generate a starter config to allow project-specific terms
typos --init

# CI-friendly machine-readable output
typos --format json
```

## Alternatives in this catalog

- [`vale`](../vale/) — prose linter with style guides; pairs with
  typos (typos for code identifiers, vale for written docs).
- [`ast-grep`](../ast-grep/) — structural code search/replace; what
  you reach for after typos when the fix needs syntax awareness.
- [`ripgrep`](../ripgrep/) — the search engine typos' speed profile
  is closest to; use rg to confirm a typo's blast radius before
  letting `--write-changes` fix it repo-wide.
