# turbo

> **Incremental, content-addressed task runner for JS/TS monorepos.**
> A single Rust binary (`turbo`) that reads a `turbo.json` task graph,
> hashes each task's inputs (source files + env + lockfile slice +
> upstream task hashes), and either replays a cached output tarball
> from local disk / a shared remote store or runs the task and uploads
> the result. Pinned to **v2.9.8** (released 2026-05-03, SPDX: `MIT`,
> [LICENSE](https://github.com/vercel/turborepo/blob/main/LICENSE)).

Source: <https://github.com/vercel/turborepo>

## TL;DR

`turbo run build test lint` walks the package graph in dependency
order, runs only the tasks whose inputs actually changed since the
last green run, and replays everything else from cache in
milliseconds. The same hash key works locally and across the team —
plug in a remote cache (Vercel, self-hosted [Turborepo Remote Cache
spec](https://turbo.build/docs/core-concepts/remote-caching), or any
S3-compatible bucket via community implementations) and the first
person who built the commit pays the cost; everyone else gets a
near-instant `FULL TURBO` replay.

## Install

```sh
# Recommended (per-repo, version pinned):
pnpm add -Dw turbo@2.9.8         # or npm i -D / yarn add -D / bun add -d

# System-wide:
brew install turbo                # macOS
npm i -g turbo@2.9.8              # any platform with Node

# Verify
turbo --version                   # 2.9.8
```

## License

MIT — unrestricted. Safe to embed in CI runner images, ship as part
of an internal monorepo bootstrap script, or vendor into a
proprietary build pipeline. The remote-cache *protocol* is also
open, so you are not locked to one cache vendor.

## Primary use case

Polyglot JS/TS monorepo (Next.js + a Vite app + a shared `ui`
package + a Node API + a `tsconfig` package + a `lint-config`
package) where `pnpm -r build` takes 4 minutes from cold but only
two of the eleven packages actually changed since the last commit.
After `turbo run build`, the changed packages rebuild and the other
nine replay from cache — wall-clock drops from 4 min to ~12 s.
Every PR's CI inherits the same speedup once a remote cache is
wired up.

## What it competes with

- **[`nx`](https://nx.dev/)** — the other dominant JS/TS monorepo
  runner. Nx is heavier (project graph, plugins, generators, an
  optional cloud DAG visualiser); `turbo` is intentionally smaller
  and config-driven (one `turbo.json`, no plugins, no codegen).
  Pick `nx` when you want generators + plugin ecosystem; pick
  `turbo` when you already have your tooling and just want
  incremental task execution + remote cache.
- **[`moon`](../moon/)** — Rust monorepo runner with similar caching
  story but broader language support (Rust, Python, Go projects in
  the same repo). `moon` is the right answer for *truly polyglot*
  monorepos; `turbo` is the right answer for JS-first monorepos
  where the JS toolchain is the bottleneck.
- **`bazel` / `pants` / `buck2`** — sound-by-construction build
  graphs with hermetic sandboxing. Order-of-magnitude more setup
  cost; correct answer at 500+ engineer scale, overkill at 5–50.
- **Plain `pnpm -r` / `yarn workspaces foreach` / `npm workspaces`**
  — no caching, no parallelism scheduling. Fine until the cold
  build crosses ~30 s; painful after that.
- **[`just`](../just/) / `make`** — task aliases without a graph.
  `turbo` is what you reach for when the alias starts to mean "ten
  packages, three task types, two of which depend on the other."

## AI-native angle

`turbo` is not an LLM tool, but it sits directly under every coding
agent that touches a JS monorepo and shapes the iteration loop:

- **Tighter agent feedback.** When [`aider`](../aider/),
  [`opencode`](../opencode/), [`claude-code`](../claude-code/), or
  [`codex`](../codex/) edits one package and re-runs `turbo run
  build test`, only the touched package + its reverse dependencies
  rebuild. The agent gets test results in seconds instead of
  minutes, which directly raises the number of edit-test cycles per
  hour and lowers token spend on "waiting" turns.
- **Deterministic hash key for prompts.** Each task hash
  (`turbo run build --dry=json`) is a stable identifier for "this
  exact code + deps + env produces this exact output." Agents can
  cite the hash in PR descriptions ("turbo cache hit on `build` ⇒
  no behavior change in `ui` package") to give human reviewers a
  cheap integrity signal.
- **Remote-cache reuse across agent runs.** A team running multiple
  parallel agent sessions (one per ticket) shares the cache: the
  first agent that builds `@acme/ui@<sha>` makes that artifact free
  for every subsequent agent on the same `<sha>`. This is a
  significant cost lever once you scale to 5–10 concurrent agent
  workers.
- **`--filter` + `--affected` for narrowed agent scope.** `turbo
  run test --filter=...[origin/main]` runs only the tests that the
  current branch's diff could have broken. Agents can use this to
  bound their own verification step instead of running the full
  suite on every edit.
- **`--dry=json` is machine-readable.** The full task graph,
  inputs, hashes, and cache status come out as JSON, so an agent
  can reason about "what would `turbo run release` actually do"
  without running it — useful for plan-then-execute prompting.

## Caveats

- **JS/TS-first.** `turbo` will *run* arbitrary commands, so a
  `python -m pytest` task works, but the input-hashing assumes
  Node-style `package.json` + lockfile semantics. For a
  polyglot-from-day-one monorepo, [`moon`](../moon/) or `bazel` is
  a better starting point.
- **Cache poisoning is real.** If a task reads an input that
  `turbo.json` does not list under `inputs` (an env var, a sibling
  package's untracked file, the system clock), the hash will be
  wrong and a stale cache hit will produce a stale output. Treat
  every task's `inputs` / `env` / `passThroughEnv` declaration as
  load-bearing; review it whenever you change what the task reads.
- **Remote-cache trust boundary.** A shared remote cache means
  anyone who can write to it can poison every downstream consumer.
  Self-hosted caches need access controls and ideally signed
  artifacts; public-internet caches should be read-only for
  unauthenticated clients.
- **2.x changed defaults vs 1.x.** If you're upgrading from 1.x,
  read the [migration
  guide](https://turbo.build/docs/crafting-your-repository/upgrading)
  — `pipeline` was renamed to `tasks`, env-var handling tightened,
  and a few flags were renamed. Pin the version in `package.json`
  and upgrade deliberately.

## Concrete example

`turbo.json` for a Next.js app + shared `ui` package:

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", "package.json", "tsconfig.json"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**"],
      "env": ["NODE_ENV", "NEXT_PUBLIC_*"]
    },
    "test": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", "test/**", "package.json"],
      "outputs": ["coverage/**"]
    },
    "lint": {
      "inputs": ["src/**", ".eslintrc*", "package.json"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

Then `turbo run build test lint` from the repo root walks the
graph, hashes every input, and either replays from cache or
executes — in topological order, with maximum parallelism subject
to dependency constraints, and with a single tidy progress UI.
