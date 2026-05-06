# hk

> **A git-hook + project-lint runner that uses read/write file
> locks to schedule linters in parallel without races** —
> configures from one `hk.pkl` file, integrates natively with
> [`mise`](../mise/) for tool resolution, and can run the same
> hook stack inside CI as it does in `pre-commit`.
> Pinned to **v1.45.0** (released 2026-05-05,
> [`gh api repos/jdx/hk/releases/latest`](https://github.com/jdx/hk/releases/latest),
> [LICENSE](https://github.com/jdx/hk/blob/main/LICENSE),
> MIT).

Source: <https://github.com/jdx/hk>

## TL;DR

Git-hook managers all solve the same surface problem ("run
`prettier` and `eslint` and `gofmt` on `pre-commit`"), and the
incumbents — [`pre-commit`](../pre-commit/) (Python),
[`lefthook`](../lefthook/) (Go), `husky` (Node) — work fine for
small repos. They diverge from `hk` on three concrete axes.
**Tool resolution**: `pre-commit` builds its own per-hook
virtualenvs; `lefthook` shells out to whatever's on `$PATH`;
`hk` integrates natively with [`mise`](../mise/) (same author),
so the linter version that runs in `pre-commit` is the same one
your editor and CI use, pinned in one file. **Concurrency
model**: `lefthook` and `pre-commit` parallelise across files
but serialise within a stage; `hk` introduces *read/write file
locks* per file so e.g. a formatter (write-lock) and a linter
(read-lock) on the same file are scheduled correctly while
formatters on *different* files run fully in parallel — measured
wins on big monorepos. **Config language**: instead of YAML it
uses [Pkl](../pkl/) (Apple's typed config language), so the
config has schema validation, IDE completion, and computed
values without YAML's anchors-and-aliases dance — the cost is
one more dependency in the bootstrap, the win is "the config
itself catches typos before the hook runs". `hk` also speaks the
`pre-commit` ecosystem (it can consume hooks defined in the
`.pre-commit-hooks.yaml` shape) so migration is incremental.

## Install

```bash
# Homebrew (macOS / Linux)
brew install hk

# via mise (same author — recommended if you already use mise)
mise use -g hk

# Cargo
cargo install hk

# Pre-built binary from a release
curl -L \
  https://github.com/jdx/hk/releases/download/v1.45.0/hk-x86_64-linux.tar.xz \
  | tar xJ && sudo mv hk /usr/local/bin/

# verify
hk --version
```

## Representative examples

```bash
# 1. Initialise a config + install the git hooks
hk init                # writes hk.pkl + .git/hooks/pre-commit, etc.

# 2. Minimal hk.pkl (run prettier + eslint on staged JS/TS)
#    hk.pkl
#    amends "package://github.com/jdx/hk/releases/download/v1.45.0/hk@1.45.0#/Config.pkl"
#    hooks {
#      ["pre-commit"] = new Hook {
#        steps {
#          ["prettier"] { glob = "*.{js,ts,tsx}"; check = "prettier --check {{files}}";
#                         fix = "prettier --write {{files}}" }
#          ["eslint"]   { glob = "*.{js,ts,tsx}"; check = "eslint {{files}}" }
#        }
#      }
#    }

# 3. Run a stage manually (CI-friendly — no git index needed)
hk run pre-commit --all
hk run pre-commit --from origin/main --to HEAD   # PR diff range

# 4. Format-only (apply fixers, skip pure linters)
hk run pre-commit --fix

# 5. Run one specific step from one stage
hk run pre-commit --step prettier

# 6. Generate the .pre-commit-config.yaml-equivalent so a repo
#    can host hk *and* still satisfy CI that already speaks
#    pre-commit
hk generate pre-commit-config > .pre-commit-config.yaml

# 7. Inside CI (GitHub Actions / GitLab / etc.)
- run: |
    curl -fsSL https://hk.jdx.dev/install.sh | sh
    hk run pre-commit --all
```

## When to use vs. alternatives

- Pick **hk** when the repo is a polyglot monorepo, the linter
  toolchain is already pinned via [`mise`](../mise/) /
  [`asdf`](../asdf/), and the win from RW-lock-aware parallel
  scheduling on hundreds of files matters (measurable on
  laptops with 8+ cores; negligible on a 50-file repo).
- Pick [`pre-commit`](../pre-commit/) when the team already
  invests in its rich ecosystem of community hooks
  (`pre-commit-hooks` repo, language-specific managers) and the
  isolated-virtualenv-per-hook story is a *feature*, not a
  cost — or when the CI runner already caches `~/.cache/pre-commit`
  and switching cache keys hurts. `hk` can consume those hooks,
  so a hybrid is fine.
- Pick [`lefthook`](../lefthook/) when YAML config and shelling
  out to `$PATH` is exactly enough — no Pkl, no read/write
  locks, smaller binary, fewer moving parts. Preferred for
  small mono-language repos where the parallelism story is
  already "good enough".
- Pick `husky` + `lint-staged` when the repo is single-language
  Node / TypeScript and the team's `npm install` step is the
  natural place to wire hooks; `husky` will always be the
  smallest delta in a JS-only stack.
- Pick a CI-only setup (no client hooks at all, just a
  `lint.yml` workflow) when the repo's PR latency tolerates
  catching issues server-side and the team finds local hooks
  intrusive — `hk` still works for the CI half if you want to
  keep config in one file.
- Caveats: requires Pkl for non-trivial configs (one extra tool
  in bootstrap — `mise` resolves it for you, but a
  fresh-clone-plus-CI run has to fetch it), the lock-graph
  scheduler is the value prop and only matters at scale (don't
  expect a win on tiny repos), and the project moves fast (1.x
  with frequent minor bumps — pin the version in `hk.pkl`'s
  `amends` line so a `brew upgrade hk` doesn't change scheduler
  behaviour mid-day).
