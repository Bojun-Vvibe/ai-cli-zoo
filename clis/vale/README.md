# vale

> **Markup-aware, rule-pack-driven prose linter** — a single
> Go binary that enforces *your* house style across Markdown,
> reStructuredText, AsciiDoc, HTML, Org, XML, and source-code
> comments via composable YAML rules. Pinned to **v3.14.1**
> (commit `41f3b223e8cc669d1309a2104e5b70df2ae6a5a8`,
> [LICENSE](https://github.com/vale-cli/vale/blob/main/LICENSE),
> MIT).

Source: <https://github.com/vale-cli/vale>

## TL;DR

`vale` is the prose-linter equivalent of ESLint or
clippy: a small fast engine plus a *style pack* (YAML
rules) that you pick or write per-project. The engine
parses each input file in its native markup and only lints
the prose nodes — code blocks, inline code, link URLs, and
front-matter pass through untouched. Rules are
composable: substitution lists ("write GitHub not
Github"), regex matches ("avoid passive voice"), readability
scores (Flesch), occurrence counts ("don't repeat 'just'
within 100 words"), and conditional checks. Five officially
maintained packs ship style guides for Microsoft (writing
style, not the org), Google, Red Hat, Joblint, and the
Proselint pack. Output is rich JSON, line-format, or a
nice colored CLI report; CI integrations exist for GitHub
Actions and pre-commit.

## Install

```bash
# Homebrew (macOS / Linux)
brew install vale

# Go install
go install github.com/vale-cli/vale@v3.14.1

# Pre-built binary from a release
curl -LO "https://github.com/vale-cli/vale/releases/download/v3.14.1/vale_3.14.1_macOS_arm64.tar.gz"
tar xf vale_3.14.1_macOS_arm64.tar.gz
sudo install vale /usr/local/bin/

# Snap
sudo snap install vale

# Docker
docker pull jdkato/vale:v3.14.1

# verify
vale --version    # vale version 3.14.1
```

Bootstrap a project: `vale ls-config` shows where vale looks
for `.vale.ini`; `vale sync` downloads any style packs
referenced in that file into `StylesPath`.

## License

MIT — see
[LICENSE](https://github.com/vale-cli/vale/blob/main/LICENSE).
Permissive; the engine binary is yours to bundle. Note that
*style packs* have their own licenses (most are MIT or
CC-BY); check each pack you import.

## One Concrete Example

```bash
# 1. one-shot lint of a single file
vale README.md
# README.md
#  3:14   warning  Consider using 'use' instead   write-good.TooWordy
#  12:1   error    Avoid 'very'                   write-good.Weasel

# 2. lint a docs tree, JSON for CI
vale --output=JSON docs/ \
  | jq '.[] | {file: .File, line: .Line, msg: .Message}'

# 3. lint with a specific output template
vale --output=line docs/  # gcc-style: file:line:col: msg

# 4. show what config vale resolved
vale ls-config

# 5. download style packs declared in .vale.ini
vale sync

# 6. limit to a single style for a quick check
vale --filter='.Severity=="error"' docs/

# 7. minimal .vale.ini for a docs repo
cat > .vale.ini <<'EOF'
StylesPath = .vale/styles
MinAlertLevel = warning
Packages = write-good, proselint
[*.md]
BasedOnStyles = Vale, write-good, proselint
EOF
vale sync && vale docs/

# 8. lint code-comment prose in a Go file
vale main.go
```

## Niche It Fills

**Per-project, per-team prose style enforcement in CI.**
Where harper / LanguageTool ship one curated rule set,
vale ships *no* rules and lets you compose what you want
from style packs. The rules live in your repo
(`.vale.ini` + `.vale/styles/`), reviewed in PRs like any
other code. That makes vale the right pick when "what is
correct" is a team decision (terminology lists, banned
words, voice and tone) rather than a universal grammar
rule.

## Why use it

Three things `vale` does that the alternatives do not:

1. **Markup-aware parsing per file type.** Markdown
   fenced code blocks, inline `code`, AsciiDoc include
   directives, RST roles, HTML `<code>` and `<pre>`, and
   YAML front-matter are recognized and skipped. You lint
   prose, not syntax.
2. **Rule packs are first-class, versioned, downloadable.**
   `vale sync` pulls the packs declared in `.vale.ini` into
   the project; CI gets the same rules as your laptop.
   Mix and match: write-good for general prose, proselint
   for clichés, your own house pack for terminology.
3. **JSON output and exit codes wire into CI cleanly.**
   GitHub Action exists; the JSON output indexes by file
   / line / severity / rule name so a CI bot can post
   inline review comments. `--minAlertLevel=error`
   distinguishes blocking issues from suggestions.

## Vs Already Cataloged

- **Vs [`markdownlint-cli2`](../markdownlint-cli2/):**
  complementary — markdownlint enforces *Markdown
  syntax* (heading levels, list indentation, link
  formatting); vale enforces *prose content* (terminology,
  voice, banned words). Both should run; they touch
  disjoint concerns.
- **Vs [`typos`](../typos/):** complementary — typos is a
  blazingly fast spelling checker tuned for code repos;
  vale handles prose grammar / style / terminology. Run
  typos for the spell pass, vale for the style pass.
- **Vs `harper` (sibling):** harper ships one curated
  English-only rule set with an LSP server for inline
  editor squiggles; vale ships *no* default rules and an
  ecosystem of versioned packs that you commit per-repo,
  with strong CI ergonomics. Pick harper if you want
  grammar feedback in your editor with zero config; pick
  vale if your team has a documented style guide that
  needs to be enforced on PRs.
- **Vs `proselint` (Python, not cataloged):** vale absorbs
  the proselint rule set as a downloadable pack
  (`Packages = proselint`) and runs it 10–50× faster as a
  single Go binary with markup awareness that proselint
  lacks.

## Caveats

- **No bundled rules.** Out of the box, `vale README.md`
  lints nothing — you must declare `Packages =` in
  `.vale.ini` and run `vale sync`. The reverse of harper's
  "works immediately" model. Plan for the bootstrap step.
- **`vale sync` requires network egress.** Style packs
  download from GitHub releases; air-gapped CI must
  vendor `.vale/styles/` into the repo and skip `sync`.
- **The original repo lived at `errata-ai/vale` and
  redirects to `vale-cli/vale`.** Old documentation,
  Homebrew taps, and Docker tags may still reference the
  old path. The MIT license and v3.x line are canonical
  at the new location.
- **No autocorrect.** Vale reports issues and references
  the rule that fired; it does not rewrite text. Couple
  with editor LSP integrations (vale-ls) if you want
  inline fix suggestions.
- **Rule authoring has a learning curve.** YAML rule
  files (`Substitution`, `Existence`, `Conditional`,
  `Occurrence`, `Readability`, etc.) are well-documented
  but each rule type has its own schema. Start by
  copying an existing pack and tweaking; do not write
  from scratch on day one.
- **JVM-style packs (LanguageTool integration) require
  external services.** Most packs are pure regex / lookup
  and ship with no runtime; a few advanced grammar packs
  call out to an external LanguageTool server. Read the
  pack README before committing.
