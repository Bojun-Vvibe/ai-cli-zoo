# moon

**Repo:** https://github.com/moonrepo/moon
**Version:** v2.2.3
**License:** MIT — [LICENSE](https://github.com/moonrepo/moon/blob/master/LICENSE)
**Language:** Rust

## What it does

`moon` is a build system and task runner purpose-built for polyglot
monorepos. It models every package as a **project** with declared
inputs, outputs, dependencies, and tasks, then runs those tasks
through a content-addressed cache, a remote cache (gRPC, REAPI-compatible),
a dependency-aware scheduler, and a per-project toolchain manager
that pins Node / Bun / Deno / Rust / Python / Go versions
reproducibly. Affected-project detection is built in: `moon ci`
diffs against a base branch and only rebuilds / retests packages
whose inputs (or transitive inputs) actually changed.

## Install

```bash
# proto (moonrepo's toolchain manager) is the recommended path
curl -fsSL https://moonrepo.dev/install/moon.sh | bash

# Homebrew
brew install moonrepo/tap/moon

# npm (so it can be a dev-dependency of the monorepo itself)
npm install --save-dev @moonrepo/cli

# verify
moon --version    # moon 2.2.3
```

## Real usage example

Bootstrap a workspace:

```bash
mkdir my-mono && cd my-mono
moon init
```

That writes `.moon/workspace.yml` and `.moon/toolchain.yml`. Add a
project at `apps/web/moon.yml`:

```yaml
language: typescript
type: application

tasks:
  build:
    command: 'tsc --build'
    inputs:
      - 'src/**/*'
      - 'tsconfig.json'
    outputs:
      - 'dist'
  test:
    command: 'vitest run'
    deps:
      - '~:build'
      - 'libs/utils:build'
```

Then:

```bash
# build only what changed since main
moon ci --base main

# run a target across the affected graph
moon run :test --affected

# explain why a task ran (or didn't)
moon query tasks --affected --json | jq '.tasks[].target'

# pin Node 20.18 + pnpm 9 reproducibly via the toolchain
moon docker scaffold web   # outputs a minimal docker context
```

## Why it's interesting (orthogonal niche)

The catalog already has `nx`-adjacent and `turbo`-adjacent ground
covered indirectly, but `moon` occupies a distinct slot: it is the
only **Rust-implemented, language-agnostic** monorepo orchestrator
here that ships its own toolchain pinning *and* speaks the
[Bazel Remote Execution API](https://github.com/bazelbuild/remote-apis)
for remote caching against any REAPI server (BuildBuddy,
NativeLink, self-hosted bazel-remote). That combination — no JVM,
no JS runtime requirement, polyglot project graph, REAPI-shaped
remote cache, deterministic toolchain — is not covered by any other
CLI in the zoo. It's the right pick when the monorepo has both a
TypeScript app and a Rust crate and a Go service and you do not
want a JVM-shaped build system to reason about all three.
