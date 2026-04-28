# volta

- **Repo**: https://github.com/volta-cli/volta
- **Version**: v2.0.2
- **License**: BSD-2-Clause (`LICENSE`)

## What it is

A Rust-based JavaScript toolchain manager that pins Node.js, npm, Yarn, and pnpm versions per-project via a `volta` block in `package.json`. When you `cd` into a project Volta intercepts the `node`/`npm`/`yarn`/`pnpm` shim and transparently invokes the pinned toolchain version, so every contributor and CI run uses the exact same toolchain without any explicit `use`/`activate` step.

## Why pick this over alternatives

Pick volta over `nvm`/`fnm`/`asdf` when you want **automatic, project-pinned switching with zero shell hooks**: there's no `nvm use` to run, no `.nvmrc` to read manually, and the pin lives committed inside `package.json` so collaborators inherit it without configuring anything.

## Install

```sh
brew install volta
volta setup
```

## Quick example

```sh
volta pin node@20.11.1
volta pin pnpm@9.1.0
node --version  # always 20.11.1 inside this project
```
