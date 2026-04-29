# bun

> **An all-in-one JavaScript runtime + toolkit written in Zig** —
> one ~50 MB native binary that fuses a JavaScriptCore-based runtime,
> a `node_modules`-compatible package manager, a bundler, a test
> runner, a TypeScript / JSX transpiler, and a `package.json` script
> runner. Pinned to **bun-v1.3.13** (released 2026-04-20),
> [LICENSE.md](https://github.com/oven-sh/bun/blob/main/LICENSE.md)
> (mixed: bun source MIT, JavaScriptCore LGPL-2.0; effective SPDX
> `NOASSERTION` per GitHub's classifier).

Source: <https://github.com/oven-sh/bun>

## TL;DR

`bun` is what you reach for when `npm install` taking 90 seconds
and `ts-node` taking another 5 seconds to boot a one-file script
have both finally crossed your tolerance threshold. The runtime
loads `.ts` / `.tsx` / `.jsx` natively (no `tsc` step, no
`ts-node`, no `tsx` wrapper), the package manager installs a
typical Next.js-shaped `node_modules` in under 5 seconds from a
warm cache, the test runner has Jest-compatible globals plus
snapshot + DOM via happy-dom built in, and the bundler produces
a single-file ESM/CJS/IIFE artifact without a `webpack.config.js`.
Drop-in for most Node scripts (`bun run script.ts` instead of
`node`/`tsx`), and the package manager works against the existing
npm registry with the same `package.json` semantics.

## Install

```bash
# macOS / Linux / WSL — official installer
curl -fsSL https://bun.sh/install | bash

# Homebrew (macOS / Linux)
brew install bun

# npm (bootstrap from an existing Node)
npm install -g bun

# Docker
docker pull oven/bun:1.3.13

# Windows (PowerShell)
# powershell -c "irm bun.sh/install.ps1 | iex"

# verify
bun --version    # 1.3.13
```

## Why catalogued

`bun` is the most credible attempt to date at collapsing the
entire JS toolchain (`node` + `npm`/`pnpm` + `tsc`/`tsx` + `jest`
+ `webpack`/`esbuild`) into one binary with no config files, and
it is fast enough that the speedup is qualitative, not just a
benchmark talking point. AI coding agents that scaffold or run
TypeScript projects (codex, opencode, claude-code, gemini-cli,
aider, crush) all benefit from `bun install` taking seconds
instead of minutes inside the agent's tool-call loop, and from
`bun run script.ts` removing the "do I need to compile this
first?" decision branch. The 1.x line is now stable enough that
`bun` can be the default shebang for a new script without
qualification.

## When to pick

- You are starting a new TypeScript / JavaScript project and
  have no legacy investment in a `webpack.config.js` or a
  `jest.config.js` you cannot rewrite.
- An AI agent is generating + running `.ts` scripts in a tight
  loop and the `node + tsx` warmup is the bottleneck.
- You need a fast `package.json` script runner (`bun run dev`)
  that boots in <50 ms vs `npm run dev`'s ~400 ms.
- You want one binary in a Docker image instead of node + npm +
  tsx + jest + esbuild.
- You want native SQLite (`bun:sqlite`), `fetch`, `WebSocket`,
  and `Bun.serve()` HTTP without `npm install`-ing them.

## When to skip

- Your project depends on a Node native addon that has not been
  ported to bun's NAPI implementation (most are fine in 1.3.x,
  but check anything that ships a `binding.gyp`).
- You need strict Node.js semantics for a package that does
  internal module-graph introspection (`require.cache` mutation,
  esoteric `Module._resolveFilename` patches).
- Your CI / production runtime is locked to a specific Node LTS
  by policy and you cannot run a second runtime alongside it.
- You need Deno-style permission sandboxing as a security
  boundary — bun deliberately does not gate filesystem / network
  by default, matching Node's posture.
- You are debugging a memory-pressure or GC issue specific to
  V8; bun uses JavaScriptCore and the heuristics differ.

## Composes well with

- [`pnpm`](https://pnpm.io/) — bun can `bun install` a pnpm
  workspace; pick whichever your team already runs.
- [`biome`](https://biomejs.dev/) — bun runs neither lint nor
  format; pair them for a Zig + Rust JS toolchain with zero
  Node dependency.
- AI coding CLIs in this catalog ([`codex`](../codex/),
  [`opencode`](../opencode/), [`claude-code`](../claude-code/),
  [`aider`](../aider/), [`crush`](../crush/)) — set `bun` as
  the runtime in generated `package.json` scripts to shave
  agent-loop latency.
