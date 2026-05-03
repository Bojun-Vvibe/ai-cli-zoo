# gofumpt

> **Stricter, opinionated superset of `gofmt` for Go.**
> A drop-in replacement for `gofmt` that adds ~25 additional
> formatting rules nobody disagrees with — collapsing redundant
> short-variable declarations, simplifying `time` arithmetic, sorting
> imports more aggressively, removing useless `else` branches after
> a `return`, normalizing octal literals, and so on. Pinned to
> **v0.9.2** (SPDX: `BSD-3-Clause`,
> [LICENSE](https://github.com/mvdan/gofumpt/blob/master/LICENSE)).

Source: <https://github.com/mvdan/gofumpt>

## TL;DR

`gofmt` is the Go community's lowest-common-denominator formatter:
deliberately conservative, never changes anything that two reasonable
Go programmers might disagree about. `gofumpt` takes the opposite
stance — it implements every formatting rule for which there is
broad community consensus but which `gofmt` declined to enforce
because it was "too opinionated."

The result is the formatter that most Go shops actually want:

- Removes empty lines at the start/end of blocks.
- Collapses `var x int = 0` → `var x int`.
- Replaces `time.Duration(x) * time.Second` → `time.Duration(x) *
  time.Second` only where idiomatic; rewrites `time.Now().Sub(t)`
  → `time.Since(t)`.
- Removes `else { return ... }` after a terminating `if` branch.
- Joins `Foo{\n\tA,\n\tB,\n}` → `Foo{A, B}` when it fits on one
  line.
- Normalizes octal literals to the `0o` prefix (`0755` → `0o755`).
- Sorts struct tags and groups standard-library imports above
  third-party.
- Plus ~20 more — full list at
  <https://github.com/mvdan/gofumpt#added-rules>.

Output is always a strict superset of `gofmt`: every file that
passes `gofumpt -d` also passes `gofmt -d`. Migration is one-way and
safe.

## Install

```sh
# macOS via Homebrew
brew install gofumpt

# Any platform via Go
go install mvdan.cc/gofumpt@v0.9.2

# Or download the prebuilt binary:
# https://github.com/mvdan/gofumpt/releases/tag/v0.9.2
```

Verify:

```sh
gofumpt -version
# v0.9.2
```

## License

BSD-3-Clause — unrestricted. Safe to bundle in editor extensions,
CI runner images, vendored `tools/` directories, or proprietary
internal SDKs.

## Concrete example: enforce gofumpt repo-wide, surface diffs in CI

```sh
# One-shot reformat of the whole tree (safe; commits cleanly).
gofumpt -w .

# CI gate: fail if any file would change.
diff_out=$(gofumpt -d .)
if [ -n "$diff_out" ]; then
  echo "gofumpt: formatting drift detected"
  echo "$diff_out"
  exit 1
fi
```

Editor wiring is a one-line config — both VS Code (Go extension
setting `"go.formatTool": "gofumpt"`) and `gopls` (`{
"gofumpt": true }` under `formatting`) speak it natively, so the
formatter runs on save without a separate watcher.

For repo-wide enforcement without breaking `gopls`-aware editors,
the canonical pattern is:

1. Set `gopls.formatting.gofumpt = true` in a checked-in
   `.vscode/settings.json` / `gopls.toml`.
2. Add a CI step that runs `gofumpt -d -l .` and fails on non-empty
   output.
3. Optionally wire `gofumpt -w` into a [`pre-commit`](../pre-commit/)
   or [`lefthook`](../lefthook/) hook for contributors who don't use
   gopls.

`golangci-lint` also bundles `gofumpt` as a linter (`enable:
- gofumpt`) — using that path means one CI process runs both
formatting and lint checks instead of two.

## Niche

`gofumpt` covers the "stricter than `gofmt`, weaker than a
prescriptive style guide" gap:

- **`gofmt`** — the Go-toolchain default; intentionally minimal.
  Many redundant constructs (`var x int = 0`, empty leading lines,
  `else { return }`) survive `gofmt` cleanly because the Go team
  decided they were a matter of taste.
- **`golangci-lint`** with style linters (`gocritic`, `revive`,
  `stylecheck`) — finds *and reports* the same issues, but does not
  reformat them. Engineers see a wall of "consider rewriting this"
  without an automated fix-up.
- **`golines`** — focused on the orthogonal problem of
  long-line wrapping; composes cleanly with `gofumpt` (run `golines`
  first, then `gofumpt -w`).

`gofumpt` is the formatter that *fixes* the issues style linters
*report*, eliminating the manual rewrite step.

## Why use it

1. **Eliminates a class of PR review comments.** "Please remove the
   blank line at the top of this function" / "you can drop the
   `else` here" / "use `time.Since` instead of `time.Now().Sub`" —
   none of these comments need to exist in any code base running
   gofumpt on save.
2. **Strict superset of `gofmt`.** Switching to `gofumpt` never
   regresses formatting; it only tightens. Rolling out across a
   monorepo is one `gofumpt -w .` commit followed by a CI gate.
3. **`gopls` integration is first-party.** The official Go language
   server has a `gofumpt: true` switch — no separate format-on-save
   tool, no editor-specific extensions, works the same in VS Code,
   Neovim, JetBrains, and Helix.
4. **Bundled in `golangci-lint`.** If the repo already runs
   `golangci-lint`, enabling the `gofumpt` linter adds zero
   processes and zero CI minutes — same binary, same cache.
5. **Same author as `sh`, `xurls`, `sh/cmd/shfmt`.** Daniel Martí
   maintains a small constellation of "stricter formatters for
   languages whose stdlib formatters declined to be opinionated";
   `gofumpt` shares the philosophy and the maintenance discipline of
   [`shfmt`](../shfmt/).

## Vs already cataloged

- **vs [`golangci-lint`](../golangci-lint/)** — golangci-lint is
  the meta-runner for Go static-analysis tools and *includes*
  `gofumpt` as one of its enableable linters. If you already run
  `golangci-lint` in CI, enable the `gofumpt` linter there instead
  of running the binary separately. If you don't, `gofumpt` as a
  standalone is ~5 MB and runs in milliseconds.
- **vs [`gofumpt`-equivalent for other languages]** —
  [`ruff`](../ruff/) format for Python, [`biome`](../biome/) for JS/
  TS, [`stylua`](../stylua/) for Lua, [`shfmt`](../shfmt/) for
  shell, [`taplo`](../taplo/) for TOML, [`yamlfmt`](../yamlfmt/) for
  YAML. Same philosophy ("opinionated, deterministic, fast"),
  different language; `gofumpt` is the Go member of that family.
- **vs [`treefmt`](../treefmt/) / [`dprint`](../dprint/)** — those
  are *meta*-formatters that orchestrate per-language formatters;
  they call `gofumpt` for `.go` files when configured. Use treefmt/
  dprint at the repo root if you have a polyglot codebase; use
  `gofumpt` directly if you only have Go.
- **vs `gofmt -s`** — `gofmt -s` is `gofmt`'s built-in
  "simplification" mode, which does ~3 of the rewrites that
  `gofumpt` does (`for _, x := range`, redundant type literals).
  `gofumpt` is a strict superset: every `gofmt -s` rewrite plus
  ~20 more. There is no reason to run `gofmt -s` once `gofumpt` is
  in the pipeline.

## Caveats

- **One-way migration only.** Once a codebase is on `gofumpt`,
  switching back to `gofmt` will produce diff churn on every file.
  Choose deliberately.
- **`gopls` users need the `gofumpt: true` setting.** Without it,
  the editor will format with `gofmt` on save and CI will reformat
  with `gofumpt`, producing a diff loop. The fix is one line of
  `gopls` config; the symptom (saved file is "wrong" by CI) is
  confusing on first encounter.
- **No configuration knobs.** `gofumpt` deliberately exposes no
  flags for "turn rule X off." If a rule is controversial enough
  that you want to disable it, the project's stance is "don't
  adopt gofumpt for this codebase." Pre-1.0 versions occasionally
  added or modified rules; v0.x is stabilizing toward 1.0.
- **`-extra` mode is unstable.** A handful of more aggressive
  rewrites (e.g., "clothe naked returns") live behind `gofumpt
  -extra` and may move in/out of that mode between minor versions.
  Pin a specific version in CI; do not run `-extra` in CI without
  also pinning.
- **Last release v0.9.2 is October 2025.** Active maintenance,
  conservative cadence; the project is approaching feature
  completeness for the rule set the maintainer is willing to
  enforce. Pin v0.9.2 and re-evaluate at the next minor.

## How `gofumpt` fits the LLM-CLI workflow

- **Agent-generated Go diffs:** running `gofumpt -w` on the
  agent's patch before commit eliminates a class of "lint failed,
  please fix" iteration loops. The agent never has to learn the
  ~25 stylistic rules; the formatter applies them after the fact.
- **Diff minimization:** because `gofumpt` is deterministic and
  whitespace-stable, agent-produced diffs reformatted by `gofumpt`
  on both sides of the change produce the smallest possible diff
  for human review — no spurious blank-line / `else` shuffling
  noise.
- **Pre-commit integration:** wire into the same hook as
  [`reviewdog`](../reviewdog/) so the agent's local commit gets
  reformatted before push, and any drift surfaces as a single PR
  comment rather than a CI failure.
- **Eval scaffolding:** "did the agent's patch pass `gofumpt
  -d`?" is a useful binary signal for code-quality eval suites,
  cheaper to compute than full `golangci-lint` and uncorrelated
  with semantic correctness.
