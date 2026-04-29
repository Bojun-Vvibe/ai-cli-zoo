# deno

> **A secure-by-default JavaScript / TypeScript / WebAssembly
> runtime built on V8 + Rust + Tokio** — one binary that runs
> `.ts` directly with no transpile step, sandboxes filesystem /
> network / env / subprocess access behind explicit `--allow-*`
> flags, ships a built-in formatter, linter, test runner,
> bundler, doc generator, LSP, and a `npm:` / `jsr:` /
> `https://` first-class import system. Pinned to **v2.7.14**
> (released 2026-04-28),
> [LICENSE.md](https://github.com/denoland/deno/blob/main/LICENSE.md)
> (MIT).

Source: <https://github.com/denoland/deno>

## TL;DR

`deno` is what you reach for when you want to run a TypeScript
file as a script — `deno run script.ts` — without first
provisioning a `package.json`, `tsconfig.json`, `node_modules/`,
and a `tsx`/`ts-node` wrapper, *and* you want the script to be
unable to read `~/.ssh/id_rsa` or `POST` to a random host
unless you grant it explicitly. The 2.x line restored full
`node:` and `npm:` compatibility, so most existing Node
libraries import directly via `import express from "npm:express"`
without a build step. The companion `jsr.io` registry is
TypeScript-native (publishes types, not just `.d.ts`) and is
the default for new Deno-first packages. Built-in tools
(`deno fmt`, `deno lint`, `deno test`, `deno bench`, `deno doc`,
`deno compile` for single-binary builds, `deno task` for
script aliases) cover the toolchain that would otherwise be
six separate npm packages.

## Install

```bash
# Official installer (macOS / Linux / WSL)
curl -fsSL https://deno.land/install.sh | sh

# Homebrew (macOS / Linux)
brew install deno

# Cargo
cargo install deno --locked

# Docker
docker pull denoland/deno:2.7.14

# Windows (PowerShell)
# irm https://deno.land/install.ps1 | iex

# verify
deno --version    # deno 2.7.14
```

## Why catalogued

`deno` is the only mainstream JS runtime that treats the
permission boundary as a first-class language-runtime concern,
which makes it the obvious choice when an AI coding agent is
asked to *execute* untrusted code it just generated. `deno run
--allow-read=./data --allow-net=api.example.com agent_step.ts`
is a one-line sandbox that does not require Docker, Firecracker,
or a separate sandbox runner. The 2.x npm/node compatibility
work also closed the "but my favourite library is on npm" gap
that kept many users on Node through the 1.x era. Native
TypeScript execution + built-in test/format/lint also removes
several agent-loop steps that would otherwise need to invoke
`tsc`, `prettier`, `eslint`, and `jest` as separate tools.

## When to pick

- You want to run a generated `.ts` script *right now* with
  no project scaffolding.
- You want the runtime itself to enforce that a script cannot
  read arbitrary files or hit arbitrary hosts (agent sandboxing,
  CI scripts, untrusted plugins).
- You are publishing a TypeScript library and want native
  `.ts` source on the registry (jsr.io) instead of shipping
  pre-compiled `.js` + `.d.ts`.
- You want `deno compile` to produce a single self-contained
  binary for distribution (no Node, no `node_modules`, no
  install step on the target machine).
- You prefer one tool for fmt + lint + test + bench + doc + LSP
  over assembling prettier + eslint + jest + typedoc.

## When to skip

- Your team's editor / CI / deployment is deeply integrated with
  Node-specific tooling (`nvm`, `pm2`, Node-native APM agents)
  and you cannot adopt a parallel runtime.
- A critical npm dependency uses Node-internal APIs that the
  `node:` shim has not implemented yet (rare in 2.x, but
  check before committing).
- You need maximum raw startup speed for tiny scripts — `bun`
  currently edges deno on cold-start microbenchmarks (deno's
  permission system + V8 add measurable overhead).
- Your shop has standardised on a single JS runtime and adding
  a second one is a governance problem larger than the speedup.

## Composes well with

- [`bun`](../bun/) — different runtime philosophy (speed-first
  vs sandbox-first); pick per-script, both can coexist.
- [`biome`](https://biomejs.dev/) — only if you want a single
  fmt/lint config that also covers a Node-side codebase; for
  pure deno, `deno fmt && deno lint` suffices.
- AI coding CLIs in this catalog ([`codex`](../codex/),
  [`opencode`](../opencode/), [`claude-code`](../claude-code/),
  [`aider`](../aider/)) — wrap their generated-script execution
  in `deno run --allow-read=. --allow-net=...` for a free
  sandbox layer that survives prompt injection of `rm -rf /`.
