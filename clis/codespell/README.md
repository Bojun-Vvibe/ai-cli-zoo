# codespell

> **Source-code-aware typo linter** that scans repositories for
> common misspellings using a curated dictionary of ~40k known
> typo→correction pairs (not a general English dictionary, so it
> doesn't false-positive on `argv`, `regex`, `frobnicate`), with
> auto-fix, per-line ignore comments, and exit-code-driven CI
> gating. Pinned to **v2.4.2**
> ([COPYING](https://github.com/codespell-project/codespell/blob/main/COPYING),
> GPL-2.0-only).

Source: <https://github.com/codespell-project/codespell>

## TL;DR

`codespell` is the right tool for "catch typos in code, comments,
and docs without dragging in a full English spell-checker that
chokes on identifiers." Its dictionary is *misspellings*, not
*words*: each entry is a known wrong spelling mapped to one or
more candidate corrections (`teh -> the`, `recieve -> receive`,
`occured -> occurred`, `seperate -> separate`). Anything not in
that dictionary is silently ignored — so `kubectl`, `serde`,
`linalg`, `mtx`, and the project's own jargon never produce false
positives. Walks files in parallel, honours `.gitignore` (with
`--builtin clear,rare,informal,usage,code,names,en-GB_to_en-US`
toggling extra dictionaries), supports per-line `# codespell:ignore
mispel` escapes, and the `-w` flag rewrites files in place with the
top suggestion. The pre-commit hook (`repo:
https://github.com/codespell-project/codespell rev: v2.4.2`) is
the standard way to wire it into a project.

## Install

```bash
# Homebrew (macOS / Linux)
brew install codespell

# pipx (recommended — isolated venv, no global pip pollution)
pipx install codespell==2.4.2

# pip
pip install codespell==2.4.2

# Debian / Ubuntu
sudo apt install codespell

# Fedora
sudo dnf install codespell

# Arch
sudo pacman -S codespell

# verify
codespell --version    # 2.4.2
```

The single binary entry point is `codespell`. No config file
required for default operation; `[tool.codespell]` in
`pyproject.toml` or a `.codespellrc` (INI) file pins
`skip = `, `ignore-words-list = `, `builtin = `, and `count` /
`quiet-level` for repeatable runs.

## License

GPL-2.0-only — see
[COPYING](https://github.com/codespell-project/codespell/blob/main/COPYING).
The dictionary files (`codespell_lib/data/*.txt`) are dual-licensed
GPL-2.0 / CC-BY-3.0 so they can be reused outside GPL contexts.
Running codespell as a CI gate or pre-commit hook does not impose
any obligation on the scanned repository — the GPL covers
distribution of codespell itself, not the projects it inspects.

## One Concrete Example

```bash
# 1. Scan a repo (defaults: walk current dir, respect .gitignore-ish heuristics)
codespell
# ./README.md:42: occured  ==> occurred
# ./src/lib.rs:118: seperate  ==> separate
# ./docs/setup.md:7: recieve  ==> receive

# 2. Auto-fix everything with the *single* most-likely correction;
#    multi-candidate typos are skipped unless --interactive.
codespell -w

# 3. Interactive triage: prompt for each multi-candidate typo
#    (3 = third suggestion, ! = skip, ? = help)
codespell -i 3

# 4. CI gate: exit non-zero on any finding, machine-readable count.
codespell --count --quiet-level 2
echo "exit=$?"

# 5. Project-specific ignores (jargon, library names that look like typos)
codespell -L crate,nd,thirdparty,ot,asend
# or in pyproject.toml:
# [tool.codespell]
# skip = "*.lock,*.svg,vendor/**"
# ignore-words-list = "crate,nd,thirdparty,ot,asend"
# builtin = "clear,rare,en-GB_to_en-US"

# 6. Scope to staged files only (useful in commit-msg / pre-commit)
git diff --cached --name-only --diff-filter=ACM | xargs codespell

# 7. Per-line escape inside source — codespell stops checking that token on that line
let mispel = "kepe";   // codespell:ignore mispel,kepe

# 8. pre-commit hook (.pre-commit-config.yaml)
# - repo: https://github.com/codespell-project/codespell
#   rev: v2.4.2
#   hooks:
#     - id: codespell
#       additional_dependencies: [tomli]   # for pyproject.toml config on py<3.11
```

## Niche It Fills

**Catch typos in source, comments, docs, and identifiers without
the false-positive flood of a real English spell-checker.** Running
`hunspell` or `aspell` over a code repo produces thousands of false
positives on identifiers (`argv`, `mtx`, `lhs`, `regex`, `frobnicate`,
domain-specific acronyms) and is unusable as a CI gate. codespell
inverts the model: it only flags *known typos* from a curated
dictionary, so the false-positive rate is essentially the rate of
`teh / the` collisions in real code (negligible). The result is a
linter that is safe to enable on day one of a repo without weeks of
custom-dictionary tuning, and that catches exactly the typos that
hurt — wrong words in error messages, swapped letters in API doc
comments, misspelled flag names.

## Why use it

Three things codespell does that the obvious alternatives do not:

1. **Misspelling-only dictionary, not a general word dictionary.**
   The ~40k entries are sourced from real-world typos found in
   open-source code (Linux kernel, Python stdlib, Debian packages,
   etc.) and curated to avoid ambiguous corrections. `aspell -c
   foo.py` produces noise; `codespell foo.py` produces signal.
   This single design choice is why codespell is the de-facto
   typo-linter in CI for projects from CPython to Kubernetes to
   the Linux kernel itself.
2. **Drop-in CI gate with stable exit codes and `--count`.** The
   non-zero exit on any finding plus the machine-readable line:col
   format means a one-line GitHub Actions step (`run: codespell
   --count`) is a working PR gate. Combined with `-w` for local
   auto-fix, the workflow is "developer pushes typo, CI fails with
   exact location, developer runs `codespell -w`, force-pushes, CI
   green" — no manual word lookup required.
3. **Per-file / per-line / per-token escape hatches at every
   layer.** When the project does have a legitimate identifier
   that looks like a typo (`crate`, `nd`, `ot`, `tread` for
   "thread" in low-level networking code), four escape mechanisms
   compose: `--skip` for path globs, `-L` / `ignore-words-list`
   for tokens, inline `// codespell:ignore` comments for one-off
   site-specific overrides, and `--ignore-regex` for whole regex
   classes (e.g. base64 strings, hex hashes). No false positive is
   ever forced to be permanent.

For documentation-heavy projects (technical books, API references,
ADRs, runbooks) codespell catches the typos that survive prose
review because the eye reads `recieve` as `receive`. Wire it once,
forget it, never publish a typo again.

## Vs Already Cataloged

- **Vs [`typos`](../typos/):** the closest sibling — a Rust
  reimplementation of the same idea (curated misspelling
  dictionary, no general English) with comparable accuracy. Pick
  `typos` when you want a single static binary with no Python
  runtime and ~10× faster scan on large monorepos. Pick `codespell`
  when (a) the surrounding ecosystem is already Python (so pip /
  pipx is the install path of least resistance), (b) you need the
  multi-candidate `-i` interactive mode (typos picks one and runs),
  (c) you want the broader and longer-curated dictionary
  (codespell's dataset has had hundreds more contributors over a
  decade). Both wire as pre-commit hooks; many repos run *both*
  for belt-and-braces coverage since their dictionaries don't
  fully overlap.
- **Vs [`shellharden`](../shellharden/) / [`shellcheck`](../shellcheck/):**
  orthogonal — those are bash *correctness* linters (quoting bugs,
  word-splitting traps); codespell is a *spelling* linter for any
  text. Compose: shellcheck + codespell as two separate pre-commit
  hooks on `*.sh` files.
- **Vs [`vale`](../vale/) (not cataloged yet):** vale is a prose
  *style* linter (passive voice, sentence length, project-specific
  terminology like "use 'sign in' not 'log in'"). codespell is a
  *typo* linter. Documentation projects often run both: codespell
  as the cheap fast gate, vale as the deeper editorial pass.
- **Vs `hunspell` / `aspell` (not cataloged):** general English
  spell-checkers — unusable on code without exhaustive
  custom-dictionary work. Use them in editor mode while writing
  prose; use codespell in CI where false-positive tolerance is
  zero.

## Caveats

- **Misspellings only — won't catch wrong-word-but-spelled-correctly
  bugs.** "than" vs "then," "affect" vs "effect," "compliment" vs
  "complement" all spell as real English words and codespell
  passes them through. For homophone-class issues, use a prose
  linter ([`vale`](https://vale.sh) or LanguageTool). Or
  augment with the `--builtin usage` and `--builtin informal`
  optional dictionaries which catch a curated subset.
- **Auto-fix (`-w`) only applies single-candidate corrections by
  default.** Multi-candidate typos (`abandonned -> abandoned,
  abandoning`) are skipped silently. Pair `-w` with a follow-up
  interactive `-i 3` pass on the same paths to triage what's left,
  and review the resulting `git diff` before committing — the
  dictionary is curated but not infallible.
- **Binary files and large generated artifacts must be excluded
  explicitly.** `--skip '*.lock,*.min.js,*.svg,*.po,vendor/**,
  node_modules/**,*.pdf,*.ipynb'` is a typical baseline. Without
  this, codespell wastes time scanning megabytes of base64 and
  occasionally false-positives on substring matches inside binary
  blobs.
- **Project-specific jargon needs an ignore list eventually.** The
  first run on any non-trivial repo surfaces 5–20 false positives
  on identifiers that happen to look like typos (`crate` looks
  like `create`, `ot` is a common networking acronym, `nd`
  appears in numpy code). Commit them to `pyproject.toml` /
  `.codespellrc` once and they stay quiet forever.
- **GPL-2.0-only on the engine.** Fine to run as a tool; a
  subtle issue if you want to embed codespell's *engine* in a
  proprietary product (the dictionary itself is dual-licensed
  CC-BY-3.0 / GPL-2.0 so it's freely reusable; the surrounding
  Python code is GPL-2.0). For "run it in CI on a private repo"
  the licence is invisible.
