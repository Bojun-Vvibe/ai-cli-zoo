# oxlint

> **Rust-native JavaScript / TypeScript linter** — a single static
> binary from the `oxc` (Oxidation Compiler) project that re-implements
> the most-used ESLint rules (≈580 rules across `eslint`,
> `typescript-eslint`, `eslint-plugin-react`,
> `eslint-plugin-react-hooks`, `eslint-plugin-jest`,
> `eslint-plugin-vitest`, `eslint-plugin-jsx-a11y`,
> `eslint-plugin-import`, `eslint-plugin-unicorn`, `eslint-plugin-n`,
> `eslint-plugin-promise`, `eslint-plugin-oxc`) on top of a
> hand-written Rust JS / TS / JSX / TSX parser and uses *file-level
> parallelism* on every CPU core. Pinned to **v1.62.0** (release
> `apps_v1.62.0` published 2026-04-27,
> [LICENSE](https://github.com/oxc-project/oxc/blob/main/LICENSE),
> MIT).

Source: <https://github.com/oxc-project/oxc> (the `apps/oxlint`
crate inside the monorepo).

## TL;DR

`oxlint` is what you reach for when `eslint` runs for 30 s on a
medium repo and 3 minutes on a large one and the lint job is the
slowest box on the CI dashboard. It is *not* a drop-in replacement
for every ESLint plugin (no plugin loader yet, no custom-rule API,
no `--fix` for every rule), but for the ~80 % of rules that are
"detect bad pattern, report range" it runs 50–100× faster on the
same tree because it does not boot Node, does not re-parse with
`@typescript-eslint/parser`, and walks the AST in parallel across
files. `oxlint .` on a 5000-file Next.js / Vite / Remix repo
typically lands in under a second on a modern laptop. Designed to
sit *in front of* ESLint in CI: run `oxlint` first as a fast gate,
let ESLint run only on PRs that pass, or replace ESLint entirely
once the rule coverage matches your existing config.

## Install

```bash
# npm / pnpm / yarn / bun (recommended; pulls the right native binary)
pnpm add --save-dev --save-exact oxlint
npx oxlint --version       # oxlint 1.62.0

# Homebrew
brew install oxlint

# Cargo
cargo install --locked oxlint

# from a release tarball
curl -L https://github.com/oxc-project/oxc/releases/download/apps_v1.62.0/oxlint-aarch64-apple-darwin.tar.gz | tar xz
sudo install oxlint /usr/local/bin/

# verify
oxlint --version           # oxlint 1.62.0
```

The npm package ships per-arch native binaries (`@oxlint/darwin-arm64`,
`@oxlint/linux-x64-gnu`, etc.) and picks the right one at install
time, so `pnpm add` plus a `package.json` script is the most
ergonomic path inside an existing JS repo.

## License

MIT — see [LICENSE](https://github.com/oxc-project/oxc/blob/main/LICENSE).
Permissive; no attribution required for shipped binaries.

## One Concrete Example

```bash
# 1. lint the whole repo, all rules at default severity
oxlint .

# 2. lint only changed files in a pre-commit hook
oxlint $(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(js|jsx|ts|tsx|mjs|cjs)$')

# 3. enable extra rule sets (TypeScript, React, jest)
oxlint --typescript-plugin --react-plugin --jest-plugin .

# 4. write a config (oxlint reads .oxlintrc.json or eslint-style config)
cat > .oxlintrc.json <<'JSON'
{
  "plugins": ["typescript", "react", "react-hooks", "unicorn"],
  "rules": {
    "no-unused-vars": "error",
    "react-hooks/exhaustive-deps": "error",
    "unicorn/no-null": "off"
  },
  "ignorePatterns": ["dist", "build", ".next", "node_modules"]
}
JSON
oxlint .

# 5. machine-readable output for CI
oxlint --format=github .   # GitHub Actions annotation format
oxlint --format=json . | jq '.diagnostics[] | {file:.filename, msg:.message}'

# 6. fail the build on any warning, like ESLint --max-warnings=0
oxlint --deny-warnings .

# 7. apply auto-fixes where supported
oxlint --fix .
```

## Niche It Fills

**ESLint-shaped lint feedback at compiler-grade speed.** ESLint's
plugin / parser / rule architecture (Node.js + dynamic require +
shared AST + per-rule visitor) is the right shape for an
*extensible* tool but the wrong shape for a *fast* tool — most
real-world ESLint configs spend the bulk of their time in the TS
parser and in plugin walk overhead, not in rule logic. `oxlint`
discards the extensibility (no third-party rules yet) and recovers
the speed: fewer process boots, no parser plugin chain, work
sharded across cores. For repos where lint is the long pole, this
collapses CI feedback from minutes to seconds.

## Why use it

Three concrete things that pay back the migration cost:

1. **50–100× faster on real codebases.** Public benchmarks (Next.js,
   VS Code, Babel) show single-digit-second `oxlint` runs against
   single-digit-minute `eslint` runs on the same files. Even at 10×
   the cost is "lint runs every save without lag" instead of "lint
   runs in CI only."
2. **Zero-config covers the common case.** With no config file
   `oxlint` enables ~120 high-signal rules (correctness,
   suspicious patterns, common React-hooks bugs, accessibility
   smells). Drop the binary in, get useful diagnostics, layer a
   config in only when defaults are wrong.
3. **`--format=github` / `--format=json` / `--format=junit` ship
   in-box.** First-class CI integration without a wrapper plugin.
   `oxlint --format=github` annotates PRs in GitHub Actions; `--format=junit`
   plugs into any test reporter; `--format=json` is one `jq` away
   from a custom dashboard.

For an LLM agent loop where "lint every patch the model writes
before applying it" is part of the inner loop, `oxlint --deny-warnings`
is a zero-startup gate that does not cost the model a coffee
break each iteration.

## Vs Already Cataloged

- **Vs [`biome`](../biome/):** closest peer — `biome` is also a
  Rust JS/TS toolchain (linter + formatter + import organiser),
  also fast, also opinionated. The split: `biome` aims to be a
  *full* ESLint + Prettier replacement with its own rule set and
  formatter; `oxlint` is *only* a linter, deliberately
  ESLint-rule-compatible (so a team already on ESLint can adopt it
  without re-learning rule names), and pairs with whatever
  formatter you already use (Prettier, `dprint`, `biome format`).
  Pick `biome` to replace ESLint *and* Prettier in one move; pick
  `oxlint` to keep your existing ESLint rule choices and just
  speed the lint phase up.
- **Vs [`dprint`](../dprint/):** orthogonal — `dprint` is a
  pluggable formatter (TS / JSON / Markdown / TOML), `oxlint` is a
  linter. They compose: `dprint fmt && oxlint .` in a pre-commit
  hook is a fast lint+format pipeline.
- **Vs [`pyrefly`](../pyrefly/):** different language, same shape —
  `pyrefly` is the Rust-native fast type-checker for Python in the
  same family of "rewrite a JS/Python tool in Rust to make CI
  cheap"; `oxlint` is the JS / TS linter half.
- **Vs [`actionlint`](../actionlint/):** different domain —
  `actionlint` checks GitHub Actions YAML, `oxlint` checks JS / TS
  source. Pair both in a single CI lint job.
- **Vs `eslint` (not cataloged):** the tool `oxlint` is replacing.
  Keep `eslint` while you depend on plugins `oxlint` does not yet
  cover (`eslint-plugin-tailwindcss`, custom in-house rules,
  `eslint-plugin-storybook`); migrate the rest. Most teams run
  both for a transition period — `oxlint` as the fast gate,
  `eslint` for the long-tail rules.

## Caveats

- **No third-party-plugin loader yet.** `oxlint` ships a fixed set
  of rule families (TS, React, jest, vitest, jsx-a11y, import,
  unicorn, promise, n, oxc); rules from `eslint-plugin-tailwindcss`
  / `eslint-plugin-storybook` / your in-house plugin do not run.
  A plugin API is on the roadmap but not stable. Audit your
  current `.eslintrc` for non-built-in plugins before treating
  `oxlint` as a 1:1 replacement.
- **`--fix` coverage is partial.** Most reported diagnostics have
  no auto-fix yet. ESLint's `--fix` will rewrite more cases.
- **Rule semantics are *mostly* identical to ESLint, not always.**
  A handful of rules have minor scope or option differences (the
  oxlint docs list these per-rule). Run both for a sprint and
  diff the report before deleting `.eslintrc`.
- **No editor-server LSP yet.** ESLint integrates into VS Code via
  `vscode-eslint`'s LSP; `oxlint` is currently CLI-only (an
  official LSP is in progress). For in-editor squiggles today you
  still want ESLint; `oxlint` shines in CI and pre-commit.
- **Versioning is fast — pin exactly.** The `oxc` monorepo
  releases multiple times per week, and rules are added /
  recategorised in nearly every release. Pin the version in
  `package.json` (`"oxlint": "1.62.0"`, not `"^1.62.0"`) and
  bump deliberately so a Wednesday minor bump does not turn
  yesterday's green CI red.
