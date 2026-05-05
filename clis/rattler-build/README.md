# rattler-build

> **Conda package builder, rewritten in Rust: faster, statically
> linked, no Python runtime required, drop-in for most
> `conda-build` recipes via a `recipe.yaml` (or legacy
> `meta.yaml`).** Pinned to **v0.64.1**
> ([LICENSE](https://github.com/prefix-dev/rattler-build/blob/main/LICENSE),
> BSD-3-Clause).

Source: <https://github.com/prefix-dev/rattler-build>

## TL;DR

`rattler-build` is the conda-package-building half of the
`rattler` ecosystem (the other half being `pixi`, the
project-level conda environment manager). It reads a
`recipe.yaml` describing how to build a `.conda` package —
sources, build steps, runtime/host dependencies, tests — and
produces a package archive ready to upload to a channel
(`conda-forge`, an internal Artifactory, `prefix.dev`). The
project's reason to exist is speed and toolchain independence:
`conda-build` is a Python application that needs a working
conda environment to bootstrap itself, which makes CI builds
slow and fragile. `rattler-build` is a single static Rust
binary you drop into a CI runner; it solves environments using
`rattler`'s native SAT solver (the same one `pixi` and `mamba`
use) instead of shelling out to `conda`.

## Install

```bash
# Pre-built binary (recommended for CI)
curl -sSfL https://github.com/prefix-dev/rattler-build/releases/download/v0.64.1/rattler-build-x86_64-unknown-linux-musl \
  -o /usr/local/bin/rattler-build && chmod +x /usr/local/bin/rattler-build

# Homebrew
brew install rattler-build

# pixi (if you already have it)
pixi global install rattler-build

# cargo
cargo install --locked --git https://github.com/prefix-dev/rattler-build --tag v0.64.1 rattler-build

# verify
rattler-build --version
```

## Examples

```bash
# Build a single recipe for the current platform
rattler-build build --recipe ./recipes/my-tool/recipe.yaml

# Build for multiple platforms in one invocation (cross-compile when toolchain allows)
rattler-build build --recipe recipe.yaml \
  --target-platform linux-64 --target-platform osx-arm64

# Convert an existing PyPI / cargo / NPM package to a conda recipe scaffold
rattler-build generate-recipe pypi requests
rattler-build generate-recipe cran ggplot2

# Test a built package (runs the recipe's `tests:` block in a fresh env)
rattler-build test --package-file ./output/linux-64/my-tool-1.2.3-h1234_0.conda

# Upload to a prefix.dev channel
rattler-build upload prefix --channel my-org my-tool-1.2.3-h1234_0.conda
```

## When to choose it

Pick `rattler-build` when you publish conda packages and your
CI build time or environment fragility has become a problem.
Concrete signals: `conda-build` invocations that take >10
minutes mostly in solver work, recipes that fail differently
in CI vs locally because the bootstrap conda env drifted, or
a need to build packages inside a distroless / Alpine container
where dragging in a full Miniconda is wasteful. Also a fit if
you are already on `pixi` for environment management — staying
in the `rattler` ecosystem keeps the solver and the package
metadata model consistent.

Skip it when your recipes lean heavily on `conda-build`
features that `rattler-build` has not implemented yet (most
common: complex `selectors:` patterns from very old recipes,
some Windows-specific build helpers). The project tracks
parity in its docs — check before migrating a large recipe
collection. Also skip if your only goal is *consuming* conda
packages; for that, `pixi` (project envs) or `mamba` (drop-in
faster `conda`) are the right shape.

## Vs adjacent tools

- **Vs `conda-build`:** the original. Python-based, Anaconda-
  maintained, widely deployed. `rattler-build` is faster,
  static, and uses the modern `recipe.yaml` schema by default
  (with a compatibility mode for `meta.yaml`). `conda-forge`
  is in the middle of a multi-year migration of feedstocks to
  `recipe.yaml`, so being native to the new format is a
  forward-looking bet.
- **Vs `boa`:** an earlier "faster conda-build" that wrapped
  `mamba`'s solver around `conda-build`'s build engine. `boa`
  is effectively superseded by `rattler-build`; new recipes
  should target `rattler-build`.
- **Vs `pixi`:** sibling project, same maintainers. `pixi`
  manages *project environments* (lockfiles, task running, dev
  shells). `rattler-build` *produces* the packages those
  environments install. They share the underlying `rattler`
  Rust crate but are separate binaries with separate concerns.
- **Vs language-native packagers (`maturin`, `setuptools`,
  `cargo`):** those produce wheels / sdists / crates for a
  single ecosystem. `rattler-build` produces conda packages,
  which solve the cross-language native-dependency problem
  (a Python package that needs a specific `libgdal`, a Rust
  binary that needs `openssl` matched to the env's `libc`).
  Use the language-native tool when the package is pure-
  language; use `rattler-build` when you ship binaries with
  native deps.

## Caveats

- **`recipe.yaml` is the new schema.** It is *not* a syntactic
  rename of `meta.yaml` — the conditional / variant model is
  cleaner but different. There is a `--legacy` mode for
  `meta.yaml` recipes; expect occasional migration work.
- **Platform support skew.** Linux x86_64 / aarch64 and macOS
  arm64/x86_64 are first-class. Windows works but historically
  has had more rough edges than `conda-build` on the same
  recipe; test before committing a full migration.
- **Solver behavior is not bit-identical to `conda-build`.**
  Same input recipe + same channel state can pick a different
  valid resolution. Usually fine, occasionally surfaces an
  upper-bound your recipe was implicitly relying on. Lock your
  build-time channel snapshot in CI rather than tracking
  `conda-forge` `latest`.
- **Pre-1.0.** v0.64.x is production-used inside `prefix.dev`
  and parts of `conda-forge`, but the CLI surface and recipe
  schema are still allowed to make breaking changes between
  minor versions. Pin the binary version in CI, read the
  release notes before bumping.
