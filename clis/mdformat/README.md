# mdformat

> **Opinionated, CommonMark-strict Markdown formatter that ships
> as a Unix-style CLI *and* a Python library, with a plugin system
> that lets you opt into GFM, frontmatter, MyST, footnotes, tables,
> table-of-contents, mkdocs admonitions, and per-language code-fence
> formatters (Black for Python fences, gofmt for Go, ruff for
> Python, prettier-like for JSON) without rebuilding the binary.**
> Pinned to **v1.0.0** (released 2025-12 on PyPI, MIT —
> [LICENSE](https://github.com/hukkin/mdformat/blob/1.0.0/LICENSE)).

Source: <https://github.com/hukkin/mdformat>

## TL;DR

`mdformat path/ --check` is the markdown equivalent of
`black --check` or `gofmt -l`: parse every `.md` under `path/` with
a real CommonMark parser ([markdown-it-py]), normalize whitespace /
list markers / emphasis style / link reference shape according to a
fixed style guide, and exit non-zero if any file would be rewritten.
Drop `--check` to write the changes. Because the round-trip is
parse → AST → re-emit (not regex), `mdformat` cannot break a
document the same way `sed`-style markdown linters routinely do —
if the input parses to a CommonMark AST, the output parses to the
same AST.

[markdown-it-py]: https://github.com/executablebooks/markdown-it-py

## Install

```sh
# pipx (recommended — isolates the env, single-purpose tool)
pipx install mdformat==1.0.0

# Inject GFM / frontmatter / table-of-contents / footnotes plugins
pipx inject mdformat mdformat-gfm mdformat-frontmatter \
                     mdformat-footnote mdformat-tables mdformat-toc

# Or one-shot via uvx (no install)
uvx mdformat==1.0.0 README.md --check

# Or pre-commit hook (the most common deployment shape)
# .pre-commit-config.yaml:
#   - repo: https://github.com/hukkin/mdformat
#     rev: 1.0.0
#     hooks:
#       - id: mdformat
#         additional_dependencies: [mdformat-gfm, mdformat-frontmatter]

# Verify
mdformat --version    # 1.0.0
```

## License

MIT — see
[LICENSE](https://github.com/hukkin/mdformat/blob/1.0.0/LICENSE).
Each plugin (`mdformat-gfm`, `mdformat-frontmatter`,
`mdformat-tables`, `mdformat-toc`, `mdformat-footnote`,
`mdformat-myst`, `mdformat-black`, `mdformat-ruff`,
`mdformat-gofmt`, `mdformat-admon`) is a separate PyPI package
under its own repo with its own (usually MIT) licence; install
only the ones you actually want active.

## Primary use case

You have a docs tree (`README.md`, `docs/**/*.md`, ADRs, runbooks)
that 8 different humans plus 3 different coding agents have edited
over a year, and the inconsistency shows: list markers flip between
`*` / `-` / `+` per file, headings drift between ATX and Setext,
trailing whitespace appears at random, code fences are
sometimes ` ``` ` sometimes `~~~`, and links rotate between inline
and reference style. You want one command that reflows everything
to one canonical shape and a CI gate that keeps it that way.
`mdformat path/ --check` is that command.

## What it competes with

- **[`prettier`](https://prettier.io) for markdown** — the JS-world
  default, and a fine choice if you already run `prettier` on
  TS/JS/CSS in the same repo. Picks different defaults (asterisks
  for emphasis, `-` for lists, no reflow), bundles markdown with a
  big Node toolchain, and is notably less strict about CommonMark
  edge cases. Pick `prettier` when markdown is one file type in a
  larger JS-formatted repo; pick `mdformat` when the markdown tree
  is the deliverable, you want CommonMark-strict round-trips, and
  you do not want a Node runtime in the critical path.
- **[`markdownlint-cli2`](../markdownlint-cli2/)** — a *linter*, not
  a formatter. It tells you "MD013/line-length exceeded" or
  "MD025/single-h1 violated"; it does not rewrite the file to fix
  most rules. Pair the two: `markdownlint-cli2` for *style policy*
  (banned constructs, prose rules), `mdformat` for *physical
  layout* (whitespace, list markers, fence style). They do not
  overlap meaningfully.
- **[`dprint`](../dprint/) with `dprint-plugin-markdown`** — single
  Rust binary, formats markdown + JSON + TOML + TS in one pass,
  much faster than `mdformat` on huge trees. Pick `dprint` when
  speed and a polyglot formatter matter more than CommonMark-strict
  parsing and a plugin ecosystem; pick `mdformat` when the
  *correctness* of the round-trip and a Python plugin API matter
  more than wall-clock.
- **[`remark` / `unified` CLI](https://github.com/remarkjs/remark)**
  — extremely flexible Node-based AST tooling, the "Babel for
  markdown." More plugins than `mdformat`, and the same
  parse-AST-reemit safety. Cost: a full Node + `remark` + plugin
  install per repo, and configuration sprawl. Pick `remark` when
  you want to *transform* markdown (rewrite links, generate TOCs
  programmatically, MDX); pick `mdformat` when you only want to
  *format* it.
- **[`mdformat-myst`](https://github.com/executablebooks/mdformat-myst)
  / [`mystmd`](https://mystmd.org)** — adjacent ecosystem for the
  MyST flavour used by Sphinx / Jupyter Book. `mdformat-myst` is a
  plugin to this same `mdformat`; `mystmd` is a separate project
  with its own CLI focused on rendering MyST documents to HTML /
  PDF, not formatting them.

## AI-native angle

Coding agents that edit docs are routinely the *worst* markdown
authors on the team — they re-flow paragraphs at 80 cols, switch
list markers mid-file, emit code fences with random language hints,
forget the trailing newline, and mix tabs into indented lists.
None of that breaks rendering on GitHub, so a human reviewer
either eats the inconsistency or hand-fixes it. `mdformat` makes
the noise structural:

- **AST-safe rewrites are agent-safe rewrites.** Because
  `mdformat` re-emits from a CommonMark AST, an agent can run
  `mdformat path/` after every edit without fear of corrupting
  content. The same cannot be said for regex-based markdown
  linters — `mdformat`'s round-trip is the safe primitive an
  agent can call as a `tool`.
- **One `--check` exit code is one CI signal.** Wire
  `mdformat --check docs/` into pre-commit and the PR check, and
  agents using [`opencode`](../opencode/) /
  [`claude-code`](../claude-code/) /
  [`codex`](../codex/) see "your markdown drifted" the same way
  they see "your tests failed" — a red X to fix before merge,
  not a wiki page no one reads.
- **Plugins are the policy surface.** Want every Python fence to be
  `black`-formatted? `pipx inject mdformat mdformat-black`. Want a
  table of contents auto-regenerated? `mdformat-toc`. The agent
  does not need to learn project-specific rules — they live in
  `pyproject.toml` `[tool.mdformat]` and the plugin list, and
  `mdformat` enforces them automatically.
- **Round-trip-stability is a fuzz target.** The project ships
  hypothesis-based tests that generate arbitrary CommonMark and
  assert `parse(format(parse(x))) == parse(x)`; agents writing
  markdown can rely on that idempotency property when reasoning
  about whether a rewrite is safe.
- **Pair with [`vale`](../vale/) / [`harper`](../harper/) /
  [`markdownlint-cli2`](../markdownlint-cli2/).** `mdformat`
  handles physical layout; the others handle prose style and
  policy. An agent that runs all four sees a complete
  "is this PR's markdown ready" picture.

## Caveats

- **It *does* re-flow text by default.** Long paragraphs are
  re-wrapped at the configured width (default: keep existing
  wrapping; `--wrap N` enforces a column). On legacy docs that
  happen to be one-line-per-paragraph, the first run will produce
  a large diff. Run `mdformat --wrap keep` if you want layout
  preserved exactly.
- **Plugins are stateful.** The set of installed `mdformat-*`
  plugins changes the output. If contributors run `mdformat` on
  their laptop without the same plugin set CI uses, they will
  produce drift. Pin both `mdformat` and every plugin in
  `pre-commit-config.yaml` `additional_dependencies` (or in the
  CI install line) so everyone formats identically.
- **Not all markdown extensions are covered.** Pandoc-flavoured
  raw-LaTeX inserts, Hugo shortcodes, Jekyll Liquid tags, and
  MDX/JSX inside markdown will pass through unchanged in the best
  case and be reflowed weirdly in the worst. For Hugo / Jekyll /
  MDX trees, test on a small sample first.
- **GFM tables only with the plugin.** Vanilla `mdformat` is
  CommonMark-strict and will *not* recognise `|` tables — they get
  treated as paragraphs. Install `mdformat-gfm` (or
  `mdformat-tables` standalone) for any repo that uses pipe tables,
  task lists, or strikethrough.
- **Python runtime cost.** `mdformat` is Python; cold start is
  ~120 ms vs. `dprint`'s ~10 ms. On a 5,000-file docs tree this
  shows up. The `--paths-from-stdin` mode plus xargs parallelism
  helps, but if formatter wall-clock dominates your CI, `dprint` is
  the faster pick.

## Concrete example

`pyproject.toml` for a docs-heavy repo:

```toml
[tool.mdformat]
wrap = 80
number = true            # 1.  / 2.  / 3.  ordered lists
end_of_line = "lf"
```

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/hukkin/mdformat
    rev: 1.0.0
    hooks:
      - id: mdformat
        additional_dependencies:
          - mdformat-gfm==0.4.1
          - mdformat-frontmatter==2.0.8
          - mdformat-footnote==0.1.1
          - mdformat-tables==1.0.0
        files: \.(md|markdown)$
```

GitHub Actions PR gate (separate from pre-commit, runs on every
push):

```yaml
- name: Format check Markdown
  run: |
    pipx install mdformat==1.0.0
    pipx inject mdformat mdformat-gfm mdformat-frontmatter
    mdformat --check docs/ README.md CHANGELOG.md
```

Result: the markdown tree has the same enforced shape that the
codebase has via `black` / `ruff` / `gofmt` — agents and humans
land identical-looking docs, and PR review can talk about content,
not whitespace.
