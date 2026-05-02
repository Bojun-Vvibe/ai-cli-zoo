# pnpm

> **Fast, disk-efficient JavaScript package manager** — installs
> a single content-addressable copy of every package version into
> a global store and hard-links it into each project's
> `node_modules`, so a fresh install of a 200-MB dependency tree
> takes seconds and ~zero extra disk after the first project on
> the machine. Strict by default: a package can only `require()`
> what it explicitly declares in `package.json`. Pinned to
> **v11.0.3**
> ([LICENSE](https://github.com/pnpm/pnpm/blob/main/LICENSE),
> MIT).

Source: <https://github.com/pnpm/pnpm>

## TL;DR

`pnpm` is the third major npm-compatible package manager
(after `npm` and `yarn`), but it differs from both in
*storage shape*: instead of giving every project its own copy
of every dependency under `node_modules/`, `pnpm` keeps one
content-addressable store under `~/.local/share/pnpm/store/v3/`
and creates a hard-linked, symlinked tree per project. The
visible `node_modules/` is mostly symlinks pointing into a flat
`.pnpm/` directory of nested real packages, so you get
deterministic dependency isolation (a package that didn't
declare `lodash` cannot `require('lodash')` even if some
sibling pulled it in) without the duplication that blew up
classic `node_modules`. `pnpm` reads the standard
`package.json`, writes its own `pnpm-lock.yaml` (cleaner diff
than `package-lock.json`), and supports workspaces, overrides,
patched dependencies, and Corepack-managed pinning out of the
box.

## Install

```bash
# Corepack (shipped with Node 16+) — pins per-repo via packageManager field
corepack enable
corepack prepare pnpm@11.0.3 --activate

# Homebrew
brew install pnpm

# Standalone script (no Node required to bootstrap)
curl -fsSL https://get.pnpm.io/install.sh | sh -

# Standalone binary from GitHub Releases
curl -fsSL -o /usr/local/bin/pnpm \
  https://github.com/pnpm/pnpm/releases/download/v11.0.3/pnpm-linuxstatic-x64
chmod +x /usr/local/bin/pnpm

# verify
pnpm --version    # 11.0.3
```

In a repo, pin the version so CI and laptops agree:

```jsonc
// package.json
{
  "packageManager": "pnpm@11.0.3"
}
```

## One Concrete Example

```bash
# fresh project
pnpm init
pnpm add react react-dom
pnpm add -D vitest @types/node typescript

# workspace (monorepo) — one root, many packages
cat > pnpm-workspace.yaml <<'YAML'
packages:
  - 'apps/*'
  - 'packages/*'
YAML

pnpm install                              # installs every workspace
pnpm --filter @acme/web add zod           # add a dep to one package
pnpm --filter @acme/web... build          # build web + its deps
pnpm -r test                              # run `test` in every workspace

# reproducible install in CI
pnpm install --frozen-lockfile            # fail if lockfile would change

# patch a published package without forking it
pnpm patch react@18.3.1
# edit files in the printed temp dir, then:
pnpm patch-commit /tmp/abc...             # writes patches/react@18.3.1.patch + pins it

# audit / outdated
pnpm audit
pnpm outdated --long
```

## Niche It Fills

**The "monorepo + reproducible CI + reasonable laptop disk
usage" sweet spot.** A 12-app workspace with overlapping
dependencies that costs `npm` 3 GB of `node_modules` per
checkout costs `pnpm` ~200 MB of project-side symlinks plus a
shared global store amortized across every checkout on the
machine. The strict resolution catches a class of bugs (`npm`'s
flat hoisting lets you `require()` packages you never
declared) at install time, not in production.

## Vs Already Cataloged

- **Vs [`yarn`](../yarn/) / npm:** all three speak
  `package.json` and roughly the same CLI verbs (`add`,
  `install`, `run`). `yarn` Berry (v3+) introduced PnP
  (Plug'n'Play) — no `node_modules` at all, dependencies
  resolved via a `.pnp.cjs` map; great in theory, painful with
  any tool that walks `node_modules` directly. `pnpm` keeps the
  `node_modules` shape (so most tooling Just Works) and wins
  on disk + strict resolution. `npm` v9+ closed much of the
  speed gap but still hoists by default.
- **Vs [`bun`](../bun/):** `bun` is a runtime + bundler +
  package manager fused into one Zig binary. As a package
  manager it's faster than `pnpm` (often 2-3×) but its
  resolver behavior and lockfile format are still moving;
  `pnpm` is the conservative choice when you want fast installs
  *without* swapping out your runtime, test runner, and bundler
  at the same time.
- **Vs [`deno`](../deno/):** `deno` does not use
  `package.json` at all — it imports from URLs / `deno.json`
  / npm specifiers. Different ecosystem; `pnpm` only enters
  the conversation if you're staying on Node.
- **Vs [`turbo`](https://turbo.build) / Nx:** those are
  *task* runners on top of a workspace; `pnpm` is the
  *package* layer underneath them. Standard stack:
  `pnpm-workspace.yaml` for layout, `turbo.json` (or
  `nx.json`) for the build graph.

## Caveats

- Strict resolution will surface latent bugs in third-party
  packages that quietly relied on hoisting — you'll see
  `Cannot find module 'X'` from packages you don't own.
  Workaround: `public-hoist-pattern[]=*` in `.npmrc` (loosens
  to npm-like behavior) or report upstream and add a targeted
  `public-hoist-pattern[]=eslint-*` line.
- The store lives in `~/.local/share/pnpm/store/v3/` (Linux),
  `~/Library/pnpm/store/v3/` (macOS), `%LOCALAPPDATA%\pnpm\store\v3\`
  (Windows). It must be on the same filesystem as your
  projects to allow hard-linking; cross-fs (e.g. project on a
  network mount, store on local disk) silently falls back to
  copies and the disk-savings argument evaporates.
- Postinstall scripts are run by default (like npm). For
  supply-chain safety in CI, set
  `enable-pre-post-scripts=false` and `side-effects-cache=false`,
  or run with `--ignore-scripts` and explicitly opt-in.
- Lockfile churn between major versions is real — `pnpm-lock.yaml`
  v6 → v7 → v9 each rewrote the format. Pin `packageManager`
  in `package.json` and let Corepack handle the upgrade
  cadence rather than letting different team members generate
  different lockfile shapes.
- Patches survive upgrades only as long as the upstream
  source files patched still exist. After a major version bump
  of the patched package, `pnpm install` will warn and you must
  rebase the patch by hand — same trade-off as
  `patch-package`.
