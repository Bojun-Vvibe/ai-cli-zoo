# biome

> **A unified Rust formatter + linter for the JS / TS / JSX / TSX /
> JSON / CSS / GraphQL world** — a single static binary that replaces
> the Prettier + ESLint + (parts of) tsserver toolchain with one
> config file, one cache, and one parser pass. Pinned to **v2.4.13**
> (commit `e31615035808fc71d47c3a8ebf1235005d999f78`,
> [LICENSE-MIT](https://github.com/biomejs/biome/blob/main/LICENSE-MIT)
> and
> [LICENSE-APACHE](https://github.com/biomejs/biome/blob/main/LICENSE-APACHE),
> dual MIT / Apache-2.0).

Source: <https://github.com/biomejs/biome>

## TL;DR

`biome` is what you reach for when your `package.json` has accreted
`prettier`, `eslint`, `eslint-config-airbnb`, half a dozen
`eslint-plugin-*`, `@typescript-eslint/parser`, and a `lint-staged` +
`husky` rig to coordinate them, and you want to delete all of it.
One Rust binary parses your code (CST-preserving, so formatter and
linter share the tree), formats it Prettier-compatibly (>97 %
identical output on real-world JS/TS), runs ~280 lint rules with
auto-fixes, and writes one `biome.json` config for the whole repo.
Format-on-save is sub-millisecond per file, full-repo lint is 10–30×
faster than the ESLint+Prettier combo it replaces, and the same
binary runs in CI (`biome ci`), Git hooks (`biome check --staged`),
and editor (`biome lsp-proxy`).

## Install

```bash
# Homebrew (macOS / Linux)
brew install biome

# npm / pnpm / yarn / bun (project-local, recommended for repo pinning)
npm install --save-dev --save-exact @biomejs/biome
pnpm add  --save-dev --save-exact @biomejs/biome
yarn add  --dev      --exact       @biomejs/biome
bun add   --dev      --exact       @biomejs/biome

# standalone binary (no Node required)
curl -L "https://github.com/biomejs/biome/releases/download/%40biomejs%2Fbiome%402.4.13/biome-darwin-arm64" -o biome
chmod +x biome && sudo mv biome /usr/local/bin/

# Linux package managers
# Arch (AUR): yay -S biome-bin
# Nix:        nix-env -iA nixpkgs.biome

# verify
biome --version    # Version: 2.4.13
```

## License

Dual MIT / Apache-2.0 — see
[LICENSE-MIT](https://github.com/biomejs/biome/blob/main/LICENSE-MIT)
and
[LICENSE-APACHE](https://github.com/biomejs/biome/blob/main/LICENSE-APACHE).
Pick whichever your project requires; permissive, no copyleft on
output, redistribute the binary freely. The same dual-license shape
as `rustfmt` / `serde` / most of the Rust ecosystem.

## One Concrete Example

```bash
# 1. scaffold biome.json (interactive — picks formatter style + lint preset)
biome init

# 2. format + lint + organize-imports + sort-keys, applying safe fixes
biome check --write .

# 3. format only (no lint), Prettier-compatibly
biome format --write src/

# 4. lint only, with unsafe fixes too (e.g. remove unused vars)
biome lint --write --unsafe src/

# 5. CI mode — read-only, machine-readable diagnostics, non-zero on issues
biome ci src/    # use this in GitHub Actions / GitLab CI / Jenkins

# 6. format only files staged in git (fast pre-commit)
biome check --staged --write

# 7. typical biome.json (commit this to the repo)
cat > biome.json <<'JSON'
{
  "$schema": "https://biomejs.dev/schemas/2.4.13/schema.json",
  "vcs":      { "enabled": true, "clientKind": "git", "useIgnoreFile": true },
  "files":    { "ignoreUnknown": true },
  "formatter": {
    "enabled": true, "indentStyle": "space", "indentWidth": 2,
    "lineWidth": 100, "lineEnding": "lf"
  },
  "linter": {
    "enabled": true,
    "rules": { "recommended": true,
               "style": { "noNonNullAssertion": "warn" },
               "correctness": { "noUnusedVariables": "error" } }
  },
  "javascript": { "formatter": { "quoteStyle": "single",
                                 "trailingCommas": "all",
                                 "semicolons": "asNeeded" } },
  "assist":     { "enabled": true,
                  "actions": { "source": { "organizeImports": "on" } } }
}
JSON

# 8. wire as an LSP for any editor (single binary, no Node)
biome lsp-proxy   # then point your editor's LSP client at this command
```

## Niche It Fills

**Replace the Prettier + ESLint + plugin sprawl with one Rust binary
and one config file.** A typical TS monorepo before `biome` carries
~30 dev-dependencies just to make formatter and linter cooperate
(prettier, eslint, eslint-config-*, eslint-plugin-import,
eslint-plugin-react-hooks, @typescript-eslint/* parser + plugin,
prettier-plugin-tailwindcss, lint-staged, husky), each with its own
config file (.prettierrc, .eslintrc.cjs, .eslintignore,
.prettierignore), each with its own cache, each parsing the same
file again. `biome` collapses that to **one** binary, **one**
`biome.json`, and **one** parser pass — and it does so without
losing the format-style your team already agreed on, because the
formatter is a deliberate Prettier-compatible reimplementation.

## Why use it

Three things `biome` does that the ESLint+Prettier stack does not:

1. **One parse, one pass, ~10–30× faster.** ESLint and Prettier each
   build their own AST per file; `biome` parses once into a CST that
   both formatter and linter consume, in parallel across CPU cores
   from a single binary. Empirically: a 50 KLOC TS monorepo that
   takes ~25 s for `eslint . && prettier --check .` runs `biome
   check .` in ~1 s on the same hardware. Format-on-save in editors
   is consistently sub-10 ms per file.
2. **Prettier-compatible formatter, no migration tax.** The formatter
   targets >97 % output identity with Prettier on real-world JS/TS,
   which means switching is usually a no-op at the diff level — your
   git history does not get a "switched formatters" megacommit
   reformatting every file. Honors `.gitignore` and `.editorconfig`
   out of the box.
3. **Zero-config editor LSP from the same binary.** `biome
   lsp-proxy` is the LSP server; point Neovim's `nvim-lspconfig`,
   VS Code's "Biome" extension, or Zed's built-in language support
   at it and you get format-on-save + lint diagnostics + quick-fix
   actions without installing any Node packages or plugin chain.
   The same binary that runs in CI runs in the editor, so "works on
   my machine, fails in CI" reduces to "we are running different
   versions" (pin via the schema URL or the npm install).

For an LLM-CLI workflow that emits TS/JS, piping through `biome
format --stdin-file-path=foo.ts` before write-out normalises quote
style, semicolons, trailing commas, and import sort order, so the
diff the human reviews is purely the semantic change the model made
— not punctuation noise the model happened to pick differently
than the codebase does.

## Vs Already Cataloged

- **Vs [`dprint`](../dprint/):** comparable scope on the formatter
  side (multi-language, Rust-backed, fast), different surface on the
  linter side (`dprint` is formatter-only and delegates lint to
  whatever you already use; `biome` is formatter + linter + assist).
  Pick `dprint` when you want one formatter to span TS + JSON +
  Markdown + TOML + Dockerfile + Rust and you are happy keeping
  ESLint or `oxc` separate; pick `biome` when JS/TS/JSON/CSS is the
  bulk of the repo and you want the linter consolidated too.
- **Vs [`ruff`](../ruff/):** the Python-side analogue. Both projects
  have the same shape (Rust binary, formatter+linter, replaces a
  whole ecosystem of legacy tools, ~10–30× speedup, single config).
  If you have a polyglot repo, `biome` for JS/TS + `ruff` for Python
  is the modern pairing; both are read-only on each other's files
  via `files.ignoreUnknown`.
- **Vs [`oxc`](../oxc/):** related project, lower-level. `oxc` is a
  collection of JS/TS Rust crates (parser, semantic analyser,
  resolver, the `oxlint` linter); `biome` is an end-user product
  with formatter + linter + assist + LSP packaged. Use `oxc`'s
  `oxlint` when you want only the linter and want the absolute
  fastest one available (~50–100× ESLint); use `biome` when you
  want the unified formatter + linter UX in one binary and one
  config.
- **Vs [`stylua`](../stylua/):** same-shape sibling for Lua. Both
  are Rust-implemented, opinionated, AST-round-trip formatters that
  ate their language ecosystem's bikeshed. `stylua` is formatter-only;
  `biome` does formatter + linter + assist on JS/TS/CSS/etc.

## Caveats

- **Vue / Svelte / Astro single-file components are not yet first-class.**
  `biome` v2 covers JS / TS / JSX / TSX / JSON / JSONC / CSS / GraphQL
  / HTML / Markdown (the last two as nightly-stability features); SFC
  formats are tracked but not production-ready as of this writing.
  Use Prettier (or framework-specific formatters) for those file
  types and let `biome` own the rest.
- **Lint rule coverage is smaller than mature ESLint config sets.**
  ~280 rules vs ESLint's thousand-plus when you count community
  plugins. `biome` ports the most-used ESLint and `typescript-eslint`
  rules first; framework-specific rule sets (e.g. `eslint-plugin-react`
  internals beyond hooks, accessibility deep-dives, `import/order`'s
  exotic options) may be missing. Migration realism: most repos lose
  ~5–10 % of their ESLint rules; few lose anything they actually
  enforced.
- **Plugin system is minimal by design.** `biome` v2 added
  GritQL-based custom rules but does not (and does not plan to)
  support arbitrary JS-defined ESLint-style plugins. If your team
  ships internal lint rules as a private npm package, you will be
  porting them to GritQL or living with them in a parallel ESLint
  invocation during the transition.
- **Configuration shape changed at v2.** v1 → v2 migration is
  scripted (`biome migrate`) but non-trivial: rule namespaces moved,
  some defaults flipped, the `assist` section split out from
  `linter`. Pin the binary version with the matching `$schema` URL
  in `biome.json` so editors and CI agree.
- **Single binary, single language frontend.** Unlike `dprint`'s
  plugin-per-language model, `biome` keeps every supported language
  inside the main binary. This is fast and consistent, but it also
  means adding a new language (e.g. Python, Ruby) is a core-team
  decision, not a community plugin you can ship yourself.
