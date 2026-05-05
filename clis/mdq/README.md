# mdq

> **`jq` for Markdown** — a single-binary Rust filter that reads a
> Markdown document on stdin (or a file argument), parses it into a
> CommonMark + GFM AST, runs a small purpose-built selector
> language over the structure (headings, lists, links, code blocks,
> tables, blockquotes, sections), and writes the matching subtree
> back out as Markdown, JSON, or plaintext. Pinned to **v0.10.0**
> (released 2026-03-22,
> [`gh api repos/yshavit/mdq/releases/latest`](https://github.com/yshavit/mdq/releases/latest),
> [LICENSE-APACHE](https://github.com/yshavit/mdq/blob/main/LICENSE-APACHE),
> Apache-2.0).

Source: <https://github.com/yshavit/mdq>

## TL;DR

Markdown is the lingua franca of READMEs, changelogs, RFCs, ADRs,
GitHub PR bodies, LLM transcripts, and team docs — but the standard
text-processing toolkit (`grep`, `sed`, `awk`) has no notion of
"the section under heading X", "every code block tagged `bash`", or
"every link in a table cell". `mdq` closes that gap by being to
Markdown what `jq` is to JSON: a streaming filter with a tiny query
language whose primitives are the actual document constructs.
Selectors look like `# Section title` (heading match), `- item`
(list item match), `[link text](url)` (link match), `` ``` lang ``
(code block by language), `:-: column header :-:` (table cell by
column), and `>` (blockquote) — they compose with `|` exactly the
way `jq` filters do, and the output format is selectable per call
(`--output md` to keep it Markdown, `--output json` to feed
downstream tools, `--output plain` to strip formatting). Single
static Rust binary, zero runtime dependencies, exit code reflects
"did the selector match anything" so it slots into shell pipelines
and CI gates.

## Install

```bash
# Cargo (any platform with a Rust toolchain)
cargo install mdq --locked

# Homebrew (macOS / Linux)
brew install mdq

# Pre-built binary from a release
curl -L \
  https://github.com/yshavit/mdq/releases/download/v0.10.0/mdq-x86_64-unknown-linux-gnu.tar.gz \
  | tar xz && sudo mv mdq /usr/local/bin/

# verify
mdq --version    # mdq 0.10.0
```

## Representative examples

```bash
# 1. Extract everything under "## Installation" from a README
mdq '# Installation' < README.md

# 2. Pull every fenced bash code block out of the docs/ tree
cat docs/*.md | mdq '```bash'

# 3. List every external link in a CHANGELOG, as JSON, for audit
mdq --output json '[](https://*)' < CHANGELOG.md \
  | jq -r '.[] | .url'

# 4. CI gate: every "## PR Notes" section must contain a checklist
mdq '# PR Notes | - [ ]' < PR_BODY.md \
  || { echo "PR notes missing checklist"; exit 1; }

# 5. Compose with jq: extract a JSON code block, then query it
mdq --output plain '```json' < spec.md | jq '.endpoints[].path'

# 6. Strip everything except the top-level summary section
mdq '# Summary' < proposal.md > summary.md
```

## When to use vs. alternatives

- Pick **mdq** when the input is Markdown and the question is
  structural ("the section under heading X", "every code block
  tagged Y", "every link in column Z of a table") — the kind of
  query `grep` / `sed` cannot express without false positives in
  prose, code blocks, and link URLs.
- Pick [`jq`](../jq/) when the input is already JSON — mdq's
  `--output json` is the bridge: shape the Markdown into JSON with
  mdq, then query with jq.
- Pick [`yq`](../yq/) when the input is YAML / TOML / XML.
- Pick [`pandoc`](../pandoc/) when the goal is *converting* the
  whole document to another format (HTML / PDF / DOCX / EPUB) —
  mdq is for *extraction*, pandoc is for *transformation* of the
  whole. Compose: `mdq '# Reference' | pandoc -o reference.html`.
- Pick [`marker`](../marker/) when the input is the wrong format
  entirely (PDF / DOCX / EPUB) and you need *Markdown out* before
  mdq can touch it.
- Pick [`htmlq`](../htmlq/) for the HTML peer of this niche — same
  idea, CSS selectors over an HTML DOM.
- Pick `rg --multiline` over Markdown when the query is genuinely
  textual ("any line containing FOO inside any heading") rather
  than structural — mdq is the *structural* answer; ripgrep stays
  the *textual* answer.
