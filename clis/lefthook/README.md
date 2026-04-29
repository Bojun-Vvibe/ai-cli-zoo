# lefthook

> **A fast, polyglot Git hooks manager — one Go binary, one
> `lefthook.yml`, parallel hook execution, file-glob filtering,
> and zero runtime dependency on Node / Python / Ruby.** Pinned
> to **v1.13.6**,
> [LICENSE](https://github.com/evilmartians/lefthook/blob/master/LICENSE),
> MIT.

Source: <https://github.com/evilmartians/lefthook>

## TL;DR

`lefthook` is what you reach for when `pre-commit` (Python),
`husky` (Node), or `overcommit` (Ruby) all force a runtime your
project should not have to install just to run a hook. It is a
single static Go binary that reads `lefthook.yml`, installs the
matching `.git/hooks/*` thin shims on `lefthook install`, and
runs the configured commands in **parallel** with file-glob
filtering (`glob: "*.{js,ts}"`), staged-file injection
(`{staged_files}`), and per-command tags + skip rules. A typical
`pre-commit` hook that runs ESLint + Prettier + tsc + test on the
staged subset goes from a 30-line `husky` + `lint-staged` setup
across two `package.json` files to a 20-line `lefthook.yml` with
no Node dependency.

## Install

```bash
# Homebrew (macOS / Linux)
brew install lefthook

# Go (any platform with a Go toolchain)
go install github.com/evilmartians/lefthook@latest

# npm / pnpm / yarn (project-local — no Node *runtime* needed at run time;
# the package ships the binary)
npm install --save-dev lefthook
pnpm add  --save-dev lefthook
yarn add  --dev      lefthook

# RubyGems / pip / Cargo / Swift Package Manager are also published
gem  install lefthook
pip  install lefthook
cargo install lefthook

# Pre-built binary
curl -L "https://github.com/evilmartians/lefthook/releases/download/v1.13.6/lefthook_1.13.6_$(uname -s)_x86_64.tar.gz" \
  | tar xz lefthook && sudo mv lefthook /usr/local/bin/

# verify + wire into a repo
lefthook version           # 1.13.6
lefthook install           # writes .git/hooks/* shims
```

## License

MIT — see
[LICENSE](https://github.com/evilmartians/lefthook/blob/master/LICENSE).
Permissive, ship the binary freely, no obligations on the hook
content you author.

## One Concrete Example

```yaml
# lefthook.yml — commit this to the repo root
pre-commit:
  parallel: true
  commands:
    eslint:
      glob: "*.{js,ts,jsx,tsx}"
      run: npx eslint --fix {staged_files}
      stage_fixed: true
    prettier:
      glob: "*.{js,ts,jsx,tsx,json,md,yml,yaml}"
      run: npx prettier --write {staged_files}
      stage_fixed: true
    tsc:
      glob: "*.{ts,tsx}"
      run: npx tsc --noEmit
    rubocop:
      glob: "*.rb"
      run: bundle exec rubocop --autocorrect {staged_files}
      stage_fixed: true
    secrets:
      run: gitleaks protect --staged --redact

commit-msg:
  commands:
    conventional:
      run: npx commitlint --edit {1}

pre-push:
  parallel: true
  commands:
    test:
      run: npm test
      tags: backend
    typecheck:
      run: npx tsc --noEmit
      tags: backend

# usage from the shell
lefthook install                   # write hooks into .git/hooks/
lefthook run pre-commit            # run the hook manually (no git involved)
lefthook run pre-push --all-files  # run against the whole tree, not just diff
LEFTHOOK=0 git commit -m "skip"    # skip all hooks for one command
lefthook uninstall                 # remove all generated shims
```

## Niche It Fills

**Polyglot, parallel, runtime-free Git hooks.** Most teams pick a
hook manager based on which runtime they already have:
`pre-commit` for Python, `husky` + `lint-staged` for Node,
`overcommit` for Ruby — which means a polyglot monorepo accretes
two or three of them. `lefthook` is one Go binary that drives
hooks for any language, runs commands in parallel by default
(serial-only `husky` + `lint-staged` is one of the slowest parts
of a typical Node pre-commit), and supports tags + skip rules
+ glob filtering + staged-file substitution natively, so a single
`lefthook.yml` covers JS + TS + Python + Ruby + Go + shell with
the same idioms.

## Why use it

Three things `lefthook` does that `husky` + `lint-staged` /
`pre-commit` do not:

1. **Parallel execution by default.** A `pre-commit` hook with 5
   commands runs all 5 simultaneously, one per CPU, with output
   interleaved by command not by line. `husky` + `lint-staged`
   runs serially unless you wire `concurrent: true` and even
   then output is harder to read; `pre-commit` (Python) runs
   serially by stage. On a typical TS monorepo, `lefthook` cuts
   pre-commit time from ~12 s to ~3 s purely from parallelism.
2. **Runtime-free single binary.** No Node runtime needed at hook
   *execution* time (the npm package just ships the platform
   binary), no Python venv, no Ruby gem dependency tree. CI
   images, Docker dev containers, and "I just cloned the repo"
   contributors all skip the "now install the hook manager's
   runtime" step.
3. **Native staged-files + glob + skip / tag DSL.** `glob:
   "*.{ts,tsx}"` filters the staged file list before the
   command runs, `{staged_files}` injects them as args,
   `stage_fixed: true` re-stages files the hook auto-fixed
   (the `lint-staged` killer feature, but native), `tags:
   [backend]` lets `lefthook run pre-push --tags backend` run
   a subset, and `skip: [merge, rebase]` skips during specific
   git states. All in one YAML file, no plugin chain.

For an LLM-CLI workflow that lives next to a repo's normal
toolchain, `lefthook` is the right place to gate "every commit
must pass `biome ci` + `ruff check` + `gitleaks protect` + the
project's secret-scrubber" without forcing `husky` + Python +
Ruby on every contributor.

## Vs Already Cataloged

- **Vs [`pre-commit`](../pre-commit/) (if cataloged):** different
  ecosystem fit. `pre-commit` (Python) has the largest
  community-curated rule library (the `pre-commit-hooks` repo
  alone has ~40 hooks, plus thousands of `repos:` entries) and
  is the de-facto standard for Python repos and many infra repos.
  `lefthook` is faster (parallel by default), runtime-free, and
  better for polyglot or Node / Ruby / Go heavy repos. Pick
  `pre-commit` when you want the catalog of community hooks; pick
  `lefthook` when you want speed and a single binary.
- **Vs [`cocogitto`](../cocogitto/):** complementary. `cocogitto`
  is conventional-commits + changelog + version-bump tooling
  (a project lifecycle tool); `lefthook` is the hook *runner*.
  A common pairing is `lefthook` driving `cog verify` on
  `commit-msg`.
- **Vs [`czkawka`](../czkawka/), [`bat`](../bat/), [`fd`](../fd/):**
  unrelated. Listed only to clarify `lefthook` is in the "git
  hook manager" niche, not the "modern Unix replacement" niche.

## Caveats

- **Hook shims are regenerated on `lefthook install`.** If you
  hand-edit `.git/hooks/pre-commit`, the next `lefthook install`
  will overwrite it. Drive *all* hook logic through
  `lefthook.yml`; if you need to extend, use `local_only` or a
  `scripts/` block calling out to a script in the repo.
- **`{staged_files}` expands to all matching files at command
  start, not per-file.** If your tool wants one-file-at-a-time
  invocation (rare these days), wrap it in a `bash -c 'for f in
  "$@"; do tool "$f"; done'` snippet — most modern tools take
  multiple files anyway.
- **Repo-local config wins over global.** A `~/.lefthookrc.yml`
  global file exists but is overridden by per-repo
  `lefthook.yml` (which is the right default — hooks should be
  versioned with the code). Don't try to enforce
  organisation-wide hooks via the global config; check
  `lefthook.yml` into the template repo instead.
- **Parallel output can be hard to read on failure.** When 5
  commands run in parallel and 2 fail, the output is interleaved
  by command and the exit summary lists which failed; for
  pinpointing the failure, re-run with `lefthook run pre-commit
  --commands eslint` to isolate. The `--no-tty` and
  `--quiet` flags help in CI.
- **Skipping hooks via env var, not `--no-verify`.** The
  documented bypass is `LEFTHOOK=0 git commit ...`; this skips
  the lefthook-managed hooks specifically, leaving any other
  hand-written hooks alone. Don't habituate to `git commit
  --no-verify` — it bypasses *all* hooks including any
  guardrails the team relies on.
