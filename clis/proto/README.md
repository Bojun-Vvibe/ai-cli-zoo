# proto

> **Pluggable, per-project polyglot toolchain version manager.**
> A single Rust binary (`proto`) that reads a `.prototools` TOML
> manifest at the repo root and installs / activates the exact
> versions of Node, Deno, Bun, Go, Rust, Python, Ruby, and any
> WASM-plugin-defined tool listed there. Pinned to **v0.52.4**
> (released 2026-04-22, SPDX: `MIT`,
> [LICENSE](https://github.com/moonrepo/proto/blob/master/LICENSE)).

Source: <https://github.com/moonrepo/proto>

## TL;DR

`proto install` walks `.prototools`, downloads the pinned version
of each tool from its upstream release channel, verifies the
checksum, and shims the binary into `~/.proto/shims/` so the right
version is on `PATH` whenever you `cd` into the repo. New tools are
added by writing or installing a small WASM plugin — no need for
the core binary to ship support for every language.

## Install

```sh
# One-line installer (downloads to ~/.proto):
curl -fsSL https://moonrepo.dev/install/proto.sh | bash -s -- 0.52.4

# Homebrew:
brew install proto

# Verify:
proto --version    # 0.52.4
```

Add `~/.proto/shims` and `~/.proto/bin` to `PATH`, then:

```sh
proto install node 22.21.1
proto install deno 2.5.0
echo 'node = "22.21.1"' >> .prototools
echo 'deno = "2.5.0"'   >> .prototools
```

## License

MIT — unrestricted. Safe to bake into developer-machine bootstrap
scripts, CI runner images, or vendor inside a closed-source
internal toolchain installer.

## Primary use case

A team where one repo is on Node 18 + pnpm 8 + Python 3.10 and
another is on Node 22 + Bun 1.2 + Python 3.12, and engineers context-
switch between them several times per day. `cd` into either repo,
the shims resolve to the right binary automatically; new hires run
`proto install` once and have both environments without a 45-minute
README of `nvm install / pyenv install / curl | bash` steps.

## What it competes with

- **[`mise`](../mise/)** — the closest competitor. `mise` is a
  Rust-rewrite descendant of `asdf` with the largest plugin
  ecosystem (`asdf` plugins work unmodified). `proto` is younger,
  uses sandboxed WASM plugins instead of arbitrary shell scripts,
  and ships a smaller core. Pick `mise` if you need an existing
  `asdf` plugin or run on platforms WASM doesn't reach; pick
  `proto` if you want sandbox-by-default and tighter integration
  with the `moon` task runner from the same author.
- **[`asdf`](../asdf/)** — the original polyglot version manager.
  Bash-based, every plugin is `curl | bash`, slow shim resolution.
  `proto` is faster, sandboxed, and statically compiled.
- **`nvm` + `pyenv` + `rbenv` + `gvm` + ...** — one tool per
  language, each with its own shim mechanism, shell hook, and
  upgrade story. `proto` collapses all of them into one manifest
  and one daemon-free shim layer.
- **[`fnm`](../fnm/) / [`volta`](../volta/)** — Node-only fast
  switchers. Better than `nvm` for Node alone; `proto` is the
  right answer when you also need Python / Go / Rust pinned in
  the same repo.
- **[`devbox`](../devbox/) / [`flox`](../flox/) / `nix`** —
  declarative environments with full hermeticity (every
  dependency, including system libs, is pinned). Stronger
  isolation than `proto`, much heavier setup. Pick those when you
  need reproducibility down to libc; pick `proto` when you just
  need "the right `node` and `python` binary on PATH per repo."

## Why include

`proto` represents the **WASM-plugin generation** of polyglot
version managers. The catalog already lists `asdf` (shell plugins),
`mise` (Rust core, Lua/shell plugins), `fnm`/`volta` (Node-only),
and `devbox`/`flox` (Nix-backed full environments). `proto` slots
between `mise` and `devbox`: lighter than Nix, sandboxed unlike
`asdf`/`mise`, and tightly integrated with the `moon` monorepo
runner so a single `.prototools` covers both the language toolchain
and the task runner version. Worth knowing about if you're
evaluating the next-generation polyglot manager landscape in 2026.
