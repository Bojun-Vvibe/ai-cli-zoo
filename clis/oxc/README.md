# oxc

> **A collection of high-performance JavaScript / TypeScript tools
> written in Rust** — a unified parser, linter, formatter, resolver,
> minifier, and transformer for the JS / TS ecosystem, with
> `oxlint` (a near-drop-in `eslint` replacement that runs 50–100×
> faster) as the flagship subcommand. Pinned to **apps_v1.61.0**
> (commit `b6dcc22d1d297465f2afc30e0388a79bf8ebdc84`,
> [LICENSE](https://github.com/oxc-project/oxc/blob/main/LICENSE),
> MIT).

Source: <https://github.com/oxc-project/oxc>

## TL;DR

`oxc` is the Rust-rewrite-of-the-JS-toolchain story for tools that
have grown too slow at modern repo sizes. The umbrella project
ships a parser (`oxc_parser`), AST + visitor (`oxc_ast`), semantic
analyzer (`oxc_semantic`), module resolver (`oxc_resolver`),
linter (`oxlint`), formatter (`oxfmt`), minifier (`oxc_minifier`),
and transformer (`oxc_transformer`) as composable Rust crates,
plus npm-distributed CLI binaries that drop into an existing JS
project without changing build config. **`oxlint`** is the entry
point most teams adopt first: `npx oxlint .` walks the repo,
runs the ~500 ported `eslint` / `eslint-plugin-{import, jsdoc,
jest, react, react-hooks, typescript, unicorn, jsx-a11y, n,
oxc, vitest, promise}` rules in parallel across all CPU cores
against the Rust-parsed AST, and finishes in seconds where
`eslint` takes minutes — a 50-file PR that cost 8 s with `eslint`
costs 0.1 s with `oxlint`, so the linter runs on every save
instead of every CI push.

## Install

```bash
# pnpm / npm / yarn / bun
pnpm add -D oxlint
npx oxlint --version

# brew (macOS / linux)
brew install oxc-project/tap/oxlint

# cargo (rust toolchain)
cargo install oxlint --locked

# direct binary (one static file)
curl -fsSL https://raw.githubusercontent.com/oxc-project/oxc/main/install.sh | sh

# verify
oxlint --version
```

Single static binary, ~10 MB, no Node runtime required for the
CLI itself (npm distribution bundles a per-platform binary).

## One Concrete Example

A monorepo `apps/web/` with TypeScript + React:

```bash
# 1. baseline lint pass with the default ruleset
cd apps/web
npx oxlint .                 # 0.4s on ~2000 files
# vs npx eslint .            # 38s on the same tree

# 2. opinionated rule selection (closer to eslint:recommended +
#    typescript-eslint/recommended + react-hooks/recommended)
npx oxlint . \
  --jsx-a11y-plugin \
  --react-plugin \
  --typescript-plugin \
  --import-plugin \
  --deny-warnings

# 3. autofix where supported (~40% of rules have safe fixes today)
npx oxlint . --fix

# 4. CI-shaped invocation: parseable JSON, fail on any finding
npx oxlint . --format json --deny-warnings > oxlint.json

# 5. Use oxc directly from a Rust build script (e.g. a custom
#    bundler) for in-process parsing without spawning the CLI
#    cargo add oxc_parser oxc_allocator oxc_span
```

## Niche It Fills

**The "Rust-rewrite-of-the-JS-toolchain" project that has shipped
production-grade replacements** for the slowest layers of the JS
ecosystem. The pattern is familiar from
[`uv`](https://github.com/astral-sh/uv) (replacing `pip`) and
[`ruff`](https://github.com/astral-sh/ruff) (replacing `flake8` /
`isort` / `black`); `oxc` is its JS / TS counterpart.
`oxlint` is the safe first adoption — it is rule-compatible with
`eslint` for the rules it has ported, can be run *alongside*
`eslint` (different config files, no clash) so you keep the long
tail of unported plugins on `eslint` while the hot path runs on
`oxlint`, and the CI cost saving alone (linter goes from a 30 s
job to a 1 s job) usually justifies the migration. The other
`oxc_*` crates are the building blocks for downstream tools
(Rolldown, Rspack, Tsdown, Rstest, the Void Zero stack) that
need a Rust-native JS / TS pipeline rather than wrapping
`@babel/parser` / `typescript`.

## Vs Already Cataloged

- **Vs [`ast-grep`](../ast-grep/):** `ast-grep` is a generic
  multi-language structural-search-and-replace tool with its own
  pattern DSL; `oxc` / `oxlint` is a JS-specific opinionated rule
  *runner* with ~500 ported rules and autofix. Use `ast-grep` for
  custom codemod-shaped queries; use `oxlint` for the standard
  eslint-shaped lint pass at speed.
- **Vs LLM-driven reviewers ([`code-review-gpt`](../code-review-gpt/),
  [`pr-agent`](../pr-agent/)):** LLM reviewers catch semantic
  bugs and style issues a deterministic linter cannot; `oxlint`
  catches the deterministic policy + syntax issues a model is
  wasteful to spend tokens on. Run both — `oxlint` first as the
  cheap gate, LLM reviewer second on what survives.
- **Vs `eslint`:** same problem space, ~50–100× the throughput
  on the rules `oxlint` has ported, but the long tail of
  `eslint` plugins (especially internal company rule packages)
  is not portable. The realistic adoption shape is "run `oxlint`
  for the ported rules + `eslint` for the unported tail" until
  the gap closes.

## Caveats

- **Rule coverage gap.** `oxlint` ports the most commonly used
  ~500 rules from `eslint` and the major plugins; the long tail
  (custom in-house plugins, niche `eslint-plugin-*` packages)
  is not yet implemented. Audit your existing `eslint` config
  before assuming a full swap is possible.
- **Autofix coverage is partial.** Roughly 40% of rules have
  safe fixers today; the rest report-only. Plan a manual cleanup
  pass for the unfixable findings on first adoption.
- **Formatter (`oxfmt`) is pre-1.0.** It targets `prettier`
  output compatibility and is fast, but the format-stability
  guarantee `prettier` users expect is not yet there — pin the
  version in `package.json` and review release notes.
- **Type-aware lint rules require a separate flow.** `oxlint`
  is a syntactic linter; for the `typescript-eslint` rules that
  need type information (`no-floating-promises`,
  `no-misused-promises`), you still need `tsc --noEmit` or the
  full `typescript-eslint` chain. The roadmap covers this via
  `oxc_type_synthesis`; not shipped at v1.61.
- **Monorepo config discovery.** `oxlint` reads `.oxlintrc.json`
  from the directory tree; nested overrides work but the lookup
  is per-file (no global `extends:` cascading from a workspace
  root by default). Pass `-c <root>/.oxlintrc.json` explicitly
  in monorepos to avoid surprises.
