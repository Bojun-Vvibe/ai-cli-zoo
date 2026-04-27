# just

> **A handy command runner — `make` without the build-system baggage.**
> A single Rust binary that reads a `justfile` (one per project, like
> a Makefile) and runs the listed recipes — no implicit file-target
> graph, no tab-vs-spaces trap, no `$(VAR)` vs `${VAR}` confusion,
> proper argument passing, and per-recipe shebang lines so a recipe
> can be Python or Node or `nu` instead of POSIX `sh`. Pinned to
> **1.50.0**
> ([LICENSE](https://github.com/casey/just/blob/master/LICENSE),
> CC0-1.0 — public-domain-equivalent).

Source: <https://github.com/casey/just>

## TL;DR

`just` is what most teams actually want when they reach for `make`
— a project-local task runner whose only job is "alias these
shell snippets behind nice names with arguments". `make`'s real
job (incremental rebuild based on file mtimes) is one most modern
projects already delegate to a real build system (`cargo`, `go
build`, `npm`, `bazel`, `cmake`); what survives in the `Makefile`
is `make test`, `make lint`, `make fmt`, `make deploy` — recipe
aliases that have nothing to do with file targets and suffer
needlessly from `make`'s syntax (tabs only, `$$` to escape `$`,
shell-per-line so `cd foo && cmd` then a new line resets cwd,
implicit rules firing when you didn't ask). `just` strips that
back to the recipe runner, fixes the syntax (spaces fine, real
multi-line shell, `set shell := ["bash", "-cu"]` to pick the
interpreter, per-recipe shebangs), and adds the things you would
have wanted — typed parameters with defaults, `just --list`
auto-help with doc comments, recipe dependencies that just chain
runs, dotenv loading, OS-conditional recipes, `just --choose`
fzf picker.

## Install

```bash
# Homebrew
brew install just

# Cargo
cargo install just

# Linux package managers
# Arch: pacman -S just
# Debian/Ubuntu (recent): apt install just
# Fedora: dnf install just
# Nix: nix-env -iA nixpkgs.just

# Windows
# scoop install just
# choco install just
# winget install Casey.Just

# from a release tarball
curl --proto '=https' --tlsv1.2 -sSf https://just.systems/install.sh | bash -s -- --to ~/.local/bin

# verify
just --version    # just 1.50.0
```

Drop a file named `justfile` (or `Justfile`, `.justfile`) at the
repo root. `just` walks up from cwd to find it, so `just test`
works from any subdirectory.

## License

CC0-1.0 (public-domain-equivalent) — see
[LICENSE](https://github.com/casey/just/blob/master/LICENSE).
No attribution required for binaries; redistribute freely.

## One Concrete Example

```just
# justfile — drop at repo root, `just --list` shows all recipes

# default recipe runs when you type bare `just`
default: lint test

# list recipes (also auto-bound to `just --list`)
help:
    @just --list

# run unit tests; depends on `build`
test: build
    cargo test --all-features

build:
    cargo build --release

# parameterised recipe with a default value
deploy env="staging":
    @echo "deploying to {{env}}"
    ./scripts/deploy.sh {{env}}

# per-recipe shebang — this recipe is a Python script, not sh
ports:
    #!/usr/bin/env python3
    import socket
    for p in (5432, 6379, 8080):
        s = socket.socket()
        ok = s.connect_ex(("127.0.0.1", p)) == 0
        print(f"{p}: {'open' if ok else 'closed'}")

# load .env automatically
set dotenv-load := true
db-shell:
    psql "$DATABASE_URL"

# OS-conditional
[macos]
open url:
    open {{url}}
[linux]
open url:
    xdg-open {{url}}
```

```bash
just                    # runs `default` → lint + test
just deploy             # → deploys to staging
just deploy production  # → deploys to production
just --list             # auto-help from doc comments
just --choose           # fzf-picker over recipes
just --evaluate         # dump variable values
just --summary          # one-line list of recipe names
```

## Niche It Fills

**The project-local task runner you check into git.** Every repo
ends up with a "how do I run the thing" cheat sheet — usually a
`Makefile` whose only `make` features used are `.PHONY`, or a
`scripts/` directory with eight bash files, or a `package.json`
`scripts:` block in a non-Node project. `just` is the small,
syntactically clean, version-pinnable answer to that whole
category: one file, one binary, no language coupling.

## Why use it

Three things `just` does that `make` does not, that pay back the
switching cost:

1. **Real argument passing with defaults and types.**
   `just deploy production` passes `production` to the recipe;
   `just deploy` uses the `staging` default. No `$(MAKEFLAGS)`
   parsing, no `make deploy ENV=production` env-var
   convention, no positional vs named confusion. Optional
   string / boolean / arg-list types.
2. **Per-recipe shebang escapes shell-of-the-day.** Drop
   `#!/usr/bin/env python3` (or `node`, or `nu`, or `pwsh`,
   or `bash -eu`) at the top of a recipe and the entire body
   runs as that interpreter — no `bash -c "$$(cat <<EOF ... EOF
   )"` heredoc gymnastics. Cross-recipe variables still
   interpolate via `{{var}}` so you can mix.
3. **`--list` is real, not a homemade `help:` target.** Doc
   comments above each recipe become the `--list` description,
   parameters and defaults render automatically, and
   `--choose` opens an fzf picker over the list — onboarding a
   new contributor is "clone the repo, run `just`".

For an LLM-CLI workflow, `justfile` is the right place to
formalise "the agent should run this command" — `just lint`,
`just test`, `just typecheck`, `just fmt-check` give the agent
a stable, project-discoverable surface that does not break when
you swap `cargo` for `bazel` or `npm test` for `pnpm test`.
Every `claude-code` / `opencode` / `codex` setup that ends up
with a `CLAUDE.md` saying "run `make test` to test" is one
`justfile` away from a real cross-language alias.

## Vs Already Cataloged

- **Vs `make` (POSIX, not cataloged):** orthogonal in theory,
  overlapping in practice. `make` is right when you genuinely
  have a file-dependency graph (`.o` from `.c`, `.html` from
  `.md`); `just` is right when every recipe is `.PHONY` and
  the file targets exist only to satisfy `make`'s model. Most
  modern repos are the second case. `just` does not implement
  incremental rebuild — keep `make` for that.
- **Vs `npm run` / `pnpm run` / `cargo xtask` (not cataloged):**
  language-locked alternatives. `npm run test` is fine in a
  Node-only repo; `cargo xtask` works in a Cargo-only repo;
  neither survives a polyglot repo (Rust backend + TS frontend
  + Python data scripts) without making one of the three
  languages canonical. `just` is language-neutral.
- **Vs `task` (Taskfile.dev, not cataloged):** closest peer.
  `task` is YAML-based with a richer dependency / file-watch
  story; `just` is a small DSL with cleaner argument
  ergonomics and no YAML indentation traps. Pick `just` when
  you want minimal syntax and per-recipe shebangs; pick `task`
  when you want file-watch and YAML config-as-data.

## Caveats

- **Not a build system.** No file-target dependency graph, no
  incremental rebuild on mtime. If you need that (compiling C,
  generating thousands of derived files), keep `make` /
  `ninja` / `bazel` underneath and have `just build` shell out
  to it.
- **Recipe dependencies always re-run.** `just test` with
  `test: build` always runs `build` first — there is no "skip
  if build is up to date" check. Push the up-to-date check
  into the underlying tool (`cargo build` is itself
  incremental).
- **`set shell` defaults to `sh -cu` on Unix and PowerShell on
  Windows.** Multi-line recipes that work on your Mac
  (interpreted as `bash`) may break on a CI runner where `sh`
  is `dash`. Either pin `set shell := ["bash", "-cu"]` at the
  top of the justfile or use per-recipe shebangs.
- **No file-watch mode.** `just --watch` does not exist; pair
  with [`watchexec`](https://github.com/watchexec/watchexec) or
  `fd -e rs | entr -c just test` for re-run-on-save.
- **Public domain (CC0) is not OSI-approved.** Some corporate
  legal review processes treat CC0 as "weird" and will ask
  questions; the practical answer is "permissive, no
  attribution, no warranty" — same shape as MIT for binary
  redistribution.
