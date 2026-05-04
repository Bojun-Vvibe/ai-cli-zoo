# knip

> **"`tsc --noUnusedLocals` for the whole repo": finds unused
> files, exports, dependencies, devDependencies, types, enum
> members, class members, duplicate exports, and unresolved imports
> in TypeScript / JavaScript projects (including monorepos), with
> ~70 framework-aware plugins so it knows that
> `next.config.js` is an entry point, that `vite.config.ts` reads
> a `vite-plugin-*` from devDependencies, and that
> `eslint.config.js` `extends:` strings resolve to npm packages
> that are *not* dead.** Pinned to **v6.11.0** (released
> 2026-04-22 on npm, ISC —
> [LICENSE](https://github.com/webpro-nl/knip/blob/knip%406.11.0/packages/knip/LICENSE)).

Source: <https://github.com/webpro-nl/knip>

## TL;DR

`npx knip` analyses `package.json`, `tsconfig.json`, every
configured plugin's config (`next.config.*`, `vite.config.*`,
`jest.config.*`, ~70 more), and the import graph rooted at the
detected entry files, then reports four categories of dead code:

1. **Unused files** — TypeScript / JavaScript files no entry path
   can reach.
2. **Unused exports / types / enum members / class members** —
   exported but never imported anywhere in the workspace.
3. **Unused dependencies / devDependencies** — listed in
   `package.json` but no source file imports them.
4. **Unlisted dependencies** — imported but missing from
   `package.json` (the bug `npm install` masks until a fresh
   clone breaks).

Default exit is non-zero on any finding, so it drops straight into
CI.

## Install

```sh
# One-shot, no install
npx knip@6.11.0

# Project install (recommended — pins the version)
npm install --save-dev knip@6.11.0
# Then add a script:
#   "scripts": { "knip": "knip" }
npm run knip

# Bun-native binary (faster cold start)
npx knip-bun@6.11.0

# Verify
npx knip --version    # 6.11.0
```

Engines: Node `^20.19.0 || >=22.12.0`. Older Node will fail to
install.

## License

ISC — see
[LICENSE](https://github.com/webpro-nl/knip/blob/knip%406.11.0/packages/knip/LICENSE).
The `knip` package is the OSS core; a separate
`@knip/...` plugin set ships under the same licence. There is no
hosted commercial product to dance around.

## Primary use case

You have a multi-year TypeScript codebase (or a turborepo /
nx / pnpm workspace monorepo) that has accumulated:

- Modules nobody imports anymore but eslint + tsc still happily
  type-check.
- `package.json` entries for libraries you stopped using two
  refactors ago, dragging in audit warnings and slowing
  `npm install`.
- Imports for libraries that work locally because your global
  cache has them but break on a fresh clone.
- "Public API" exports that no consumer in the workspace actually
  consumes.

`tsc --noUnusedLocals` cannot see across files; `eslint
no-unused-vars` cannot see across packages; `depcheck` is a much
narrower tool focused only on `package.json` dead deps. `knip`
covers all four categories under one config and one CLI.

## What it competes with

- **[`depcheck`](https://github.com/depcheck/depcheck)** — the
  long-standing dead-`package.json` tool. Narrower scope (does not
  find unused files / exports / class members), and slower-moving
  parser (no first-class TS5 / monorepo / oxc support). Pick
  `depcheck` for tiny single-package projects where adding
  configuration to `knip` is overkill; pick `knip` for anything
  serious.
- **[`ts-prune`](https://github.com/nadeesha/ts-prune)** — finds
  unused exports only, project archived since 2023. `knip` is the
  modern successor and a strict superset.
- **`tsc --noUnusedLocals` /
  `eslint-plugin-unused-imports`** — file-local only. Catch
  per-file dead bindings, miss every cross-file dead module and
  every dead `package.json` entry. Run them *and* `knip`.
- **[`cargo-machete`](../cargo-machete/) /
  [`cargo-udeps`](https://github.com/est31/cargo-udeps)** — the
  Rust analogues, narrower (deps only, no dead-export analysis),
  obviously language-specific. `knip` is the JS/TS-side member of
  the same family.
- **[`madge`](https://github.com/pahen/madge) /
  [`dependency-cruiser`](https://github.com/sverweij/dependency-cruiser)**
  — produce import graphs and orphan-file lists, but do not analyse
  `package.json`, exports, or class members. Use them for
  visualisation; use `knip` for the unified pass.
- **[`oxlint`](../oxlint/) +
  [`biome`](../biome/)** — fast Rust-based linters, focused on lint
  rules rather than dead-code-graph analysis. Orthogonal — run
  `oxlint` for per-file lint speed, `knip` for whole-repo dead
  code. Notably, `knip@6` itself is built on `oxc-parser` /
  `oxc-resolver` for parsing speed.

## AI-native angle

Coding agents *generate* dead code at a depressing rate: they
stub out helper functions "for completeness," scaffold
`package.json` deps they end up not using, leave half-finished
test files in the tree, and re-export symbols nobody consumes.
Without a `knip`-shaped check, the next agent run sees the
detritus as part of the "real" codebase and learns from it.

- **Closes the agent feedback loop on tree shape.** An agent
  finishes a feature, runs `knip`, sees "the helper you added
  isn't imported anywhere" and either deletes it or wires it up.
  Without that check, dead code accumulates each iteration.
- **`--reporter json` is the agent surface.** `knip --reporter
  json --no-exit-code` emits a structured `{files: [...], issues:
  {types: {file: [{symbol, line, col}], ...}}}` document an agent
  can ingest, prioritise, and either auto-delete or open a PR
  per category.
- **Configuration is small and project-relative.**
  `knip.config.ts` typically fits on a screen — `entry`, `project`
  globs, plugin overrides. An agent onboarded to a repo can read
  the config and understand the dead-code policy in one prompt.
- **Pair with [`patchwork`](../patchwork/) auto-fix flows.**
  "Run `knip --reporter json`, for each unused export emit a
  `git rm` or `unexport` patch, run the test suite, open a PR if
  green" is a standard `Patchflow`-shaped pipeline. The hard
  part (finding the dead code) is `knip`'s job; the LLM just
  decides what to do with it.
- **Polyglot completeness.** Pair `knip` (JS/TS) with
  [`cargo-machete`](../cargo-machete/) (Rust),
  [`unimport`](https://github.com/hadialqattan/unimport) (Python
  imports), and `goimports -w -local` (Go) and you have one
  dead-code check per language an agent can run before opening
  any PR.

## Caveats

- **Configuration is required for non-trivial repos.** Out of the
  box, `knip` is conservative — it will report everything not
  reachable from a few standard entry points (`bin`, `main`,
  `module`, `exports`, common config files). For a repo that uses
  unusual entry points (server-side renderers, Rust/Go-driven
  bundlers, dynamic glob-based routing), you will need to tune
  `entry` patterns or it will scream about "unused" application
  code on the first run.
- **Plugins matter.** ~70 first-class plugins cover Next.js, Nuxt,
  Vite, Vitest, Jest, Cypress, Playwright, Storybook, ESLint,
  Stylelint, Tailwind, Astro, Remix, SvelteKit, Solid, Qwik,
  Gatsby, NestJS, etc. If your framework lacks a plugin, expect
  false positives on framework-magic files until you add custom
  `entry` patterns.
- **Dynamic imports and string-keyed lookups are blind spots.**
  `import(`./pages/${name}`)`, registry-pattern code, and
  metaprogramming-heavy DI containers will look "unused" to the
  static graph. Add `ignore` patterns for those subtrees rather
  than chasing every false positive.
- **Monorepo semantics need attention.** In a pnpm/turborepo
  workspace, "is `lodash` an unused dep of package A?" depends on
  whether package A actually imports it directly vs. transitively
  through package B. `knip` understands workspaces and supports
  per-workspace and global-root config; expect to spend an hour
  on the first config for a real monorepo.
- **First run is loud.** A real codebase that has never run
  `knip` will produce hundreds of findings. Triage in batches by
  category, not all at once; commit `--include` or `ignore`
  filters as the policy ratchets up.

## Concrete example

`knip.config.ts` for a Next.js + tRPC + Prisma monorepo, scoped
to one workspace:

```ts
import type { KnipConfig } from "knip";

const config: KnipConfig = {
  workspaces: {
    "apps/web": {
      entry: [
        "next.config.{js,ts}",
        "src/app/**/{page,layout,loading,error,not-found,route}.{ts,tsx}",
        "src/middleware.ts",
        "src/instrumentation.ts",
        "scripts/**/*.ts",
      ],
      project: ["src/**/*.{ts,tsx}", "scripts/**/*.ts"],
      next: { config: ["next.config.ts"] },
      ignoreDependencies: [
        // Tailwind picks these up via PostCSS, not via JS imports
        "tailwindcss",
        "@tailwindcss/postcss",
      ],
    },
    "packages/db": {
      entry: ["src/index.ts", "prisma/seed.ts"],
      project: ["src/**/*.ts", "prisma/**/*.ts"],
    },
  },
};

export default config;
```

CI gate:

```yaml
- name: Knip (dead code / deps)
  run: |
    pnpm install --frozen-lockfile
    pnpm exec knip --reporter compact --include files,dependencies,exports
```

Result: every PR sees a red X if it adds an unused file, an
unused export, or an unused dependency, and a green check
otherwise. Combined with `tsc --noEmit`, `eslint`, `vitest`, and
[`semgrep`](../semgrep/), the bar for "is this PR safe to merge"
is consistent across humans and agents.
