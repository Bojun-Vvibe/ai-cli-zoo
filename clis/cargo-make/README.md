# cargo-make

> **Cross-platform Rust task runner driven by a TOML `Makefile.toml`** —
> a `cargo` subcommand that defines tasks (with dependencies, env,
> conditions, platform overrides, script blocks in shell / duckscript
> / shebang) and runs them in dependency order, replacing the role
> that `make` / `just` / shell-script salad usually plays around a
> Cargo workspace. Pinned to **v0.37.24** (release published
> 2025-01-18,
> [LICENSE](https://github.com/sagiegurari/cargo-make/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/sagiegurari/cargo-make>.

## TL;DR

`cargo-make` is what you reach for when `cargo build && cargo test
&& cargo clippy && cargo fmt --check && cargo audit && cargo deny
check && cargo doc --no-deps && cargo bench --no-run` is the *real*
build and you keep re-typing it (or scripting it in a `make` recipe
that breaks on Windows). It defines a task graph in `Makefile.toml`,
each task can declare `dependencies`, `env`, `condition`,
`platform`-specific overrides, and a `script` block written in
shell, PowerShell, Python, or duckscript (the bundled
cross-platform scripting language). It ships with a *predefined*
task set covering the entire idiomatic Rust workflow — `cargo make
ci` runs build + test + clippy + fmt + audit on Linux / macOS /
Windows out of the box without any local config — and lets you
override or extend any of those tasks in your project's
`Makefile.toml`.

## Install

```bash
# cargo
cargo install --locked cargo-make

# cargo-binstall (faster, prebuilt)
cargo binstall cargo-make

# Homebrew
brew install cargo-make

# from a release tarball
curl -L https://github.com/sagiegurari/cargo-make/releases/download/0.37.24/cargo-make-v0.37.24-aarch64-apple-darwin.zip \
  -o cargo-make.zip && unzip cargo-make.zip && \
  sudo install cargo-make /usr/local/bin/

# verify
cargo make --version    # cargo-make 0.37.24
```

`cargo-make` installs two binaries: `cargo-make` (invoked as
`cargo make <task>`) and `makers` (a standalone entry point for
non-Cargo projects, e.g. running tasks in a Python or Go repo
that just wants the task graph).

## License

Apache-2.0 — see [LICENSE](https://github.com/sagiegurari/cargo-make/blob/master/LICENSE).
Permissive; ship the binary and any task scripts you author
without attribution constraints.

## One Concrete Example

```toml
# Makefile.toml — at repo root

[env]
CARGO_TERM_COLOR = "always"
RUST_BACKTRACE   = "1"

[tasks.fmt]
description = "Format every crate."
command     = "cargo"
args        = ["fmt", "--all"]

[tasks.fmt-check]
description = "Verify formatting without rewriting."
command     = "cargo"
args        = ["fmt", "--all", "--", "--check"]

[tasks.lint]
description  = "Clippy with our deny-set."
command      = "cargo"
args         = ["clippy", "--all-targets", "--all-features", "--", "-D", "warnings"]
dependencies = ["fmt-check"]

[tasks.test]
description  = "Run the full test matrix via nextest."
command      = "cargo"
args         = ["nextest", "run", "--all-features"]
dependencies = ["lint"]

[tasks.audit]
description = "Vulnerability scan of the dependency tree."
command     = "cargo"
args        = ["audit", "--deny", "warnings"]

[tasks.docs]
description = "Generate docs and fail on broken intra-doc links."
command     = "cargo"
args        = ["doc", "--no-deps", "--all-features"]
env         = { RUSTDOCFLAGS = "-D rustdoc::broken-intra-doc-links" }

[tasks.ci]
description  = "What CI runs."
dependencies = ["fmt-check", "lint", "test", "audit", "docs"]

[tasks.release-tag]
description = "Cross-platform release tag — duckscript means this works on Windows too."
script_runner = "@duckscript"
script = '''
version = exec --get-stdout cargo pkgid
echo Releasing ${version}
exec git tag -a v${version} -m "release ${version}"
exec git push origin v${version}
'''

[tasks.bench]
description       = "Run criterion benches; only on x86_64 Linux."
condition         = { platforms = ["linux"], rust_version = { min = "1.75.0" } }
command           = "cargo"
args              = ["bench", "--all-features"]
```

```bash
# everything CI runs, in dependency order, in one command
cargo make ci

# just the lint chain (which transitively runs fmt-check)
cargo make lint

# list every task with its description
cargo make --list-all-steps

# dry-run the dependency graph without executing
cargo make --print-steps ci
```

## Niche It Fills

**The cross-platform task graph that `cargo` itself does not
ship.** `cargo` knows how to build / test / lint a single command;
it does not know how to *compose* those commands into a workflow,
declare environment variables per task, run platform-specific
fallbacks, or guard a task on a Rust version / toolchain / git
branch. The usual answers are (a) a `Makefile` (Unix-only,
makefile syntax), (b) a `justfile` (cross-platform but no
dependencies-with-conditions, no built-in Rust task library),
(c) a pile of shell scripts (works until Windows). `cargo-make`
fills the gap with TOML syntax that Rust developers already read
fluently, a real dependency graph, conditional task gating, and
*~200 predefined tasks* covering the Cargo ecosystem (clippy,
fmt, audit, deny, outdated, doc, bench, coverage with grcov /
tarpaulin, cross compilation, publish, changelog generation).

## Why use it

Three concrete things that pay back the `Makefile.toml` adoption:

1. **`cargo make ci` is one line, runs everywhere.** The bundled
   default `Makefile.toml` (auto-loaded if you do not provide one)
   already implements `ci`, `build`, `test`, `coverage`,
   `audit-flow`, `outdated-flow`, `pre-publish`, `publish`, etc.
   You only write a `Makefile.toml` to *override* defaults, not to
   re-implement them. A brand-new repo gets a usable task graph for
   free.
2. **Conditions and platform overrides are first-class.**
   `condition = { platforms = ["windows"], env_set = ["CI"] }` on
   a task gates it without an `if` ladder. Per-platform
   `[tasks.foo.linux]` / `[tasks.foo.windows]` blocks override
   `command`/`args` per OS. The same `Makefile.toml` runs
   identically on a developer's macOS laptop, a Linux GitHub
   Actions runner, and a Windows Azure Pipelines agent.
3. **Duckscript means scripts are portable without a shell.** When
   you do need a procedural step (parse a version, generate a
   file, conditionally tag a release), `script_runner = "@duckscript"`
   gives you a small but real scripting language built into
   `cargo-make` itself — no `bash` / `pwsh` / `python` dependency
   on the host. The script that publishes your release runs the
   same on Windows CI as on a Mac.

For an LLM agent that drives a Rust repo's lifecycle ("format,
lint, test, then commit"), `cargo make <task>` is a stable
interface: the model targets task names, not raw cargo flags, so
project-specific quirks (extra features, custom audit policies,
non-default test runners) live in `Makefile.toml` instead of in
the prompt.

## Vs Already Cataloged

- **Vs [`just`](../just/):** closest peer in the "modern task
  runner" niche. The split: `just` is language-agnostic, has a
  small bespoke recipe syntax, and treats every recipe as an
  independent shell invocation; `cargo-make` is Rust-flavored
  (auto-detects Cargo workspaces, ships predefined Rust tasks,
  reads `Cargo.toml` metadata), uses TOML, and has a real
  dependency graph with conditions. Pick `just` for a polyglot
  monorepo where most tasks are one-line shell commands; pick
  `cargo-make` for a Rust workspace where tasks compose, gate on
  platform, and you want the bundled Cargo-ecosystem task
  library.
- **Vs [`mage`](../mage/):** different language, same role —
  `mage` writes tasks in Go and is the natural pick for Go
  projects; `cargo-make` writes tasks in TOML (+ shell /
  duckscript) and is the natural pick for Rust projects. Same
  niche, partitioned by host language.
- **Vs [`task`](../task/):** closest non-Rust peer. `task` is a
  YAML-based cross-platform task runner with a very similar
  feature set (deps, conditions, env, platform overrides). It
  has no Rust-specific task library and uses YAML rather than
  TOML. For a polyglot team that wants one task runner across
  all repos, `task` is the more neutral choice; for a Rust shop
  that lives in `Cargo.toml` already, `cargo-make` keeps the
  config-file dialect consistent.
- **Vs [`bacon`](../bacon/):** orthogonal — `bacon` is a *background
  watcher* that re-runs `cargo check` / `clippy` / `test` on file
  save; `cargo-make` is the *task definition layer*. Pair them:
  `bacon` runs `cargo make watch-test` on every save.
- **Vs [`cargo-nextest`](../cargo-nextest/) / [`cargo-mutants`](../cargo-mutants/) /
  [`cargo-deny`](../cargo-deny/):** orthogonal — these are *the
  tools `cargo-make` orchestrates*. A typical `Makefile.toml`
  invokes `cargo nextest`, `cargo deny`, `cargo mutants` as task
  commands.

## Caveats

- **Bundled defaults are opinionated.** The implicit
  `Makefile.toml` enables a lot of tasks (formatter, clippy,
  audit, outdated, doc, coverage). On a clean machine `cargo
  make ci` may fail because the underlying tools (`cargo audit`,
  `cargo outdated`) are not installed; either `cargo install` them
  up front or override the offending tasks in your project's
  `Makefile.toml`.
- **TOML grows quickly.** A non-trivial workflow with
  per-platform overrides, env injection, and multiple script
  runners produces a `Makefile.toml` of a few hundred lines.
  Use `[tasks.foo.extend]` / `extend` at the file level to split
  large configs into included files (`extend = "ci.toml"`).
- **Two execution modes can confuse.** `command` + `args` runs a
  process; `script` runs a script block. They are mutually
  exclusive per task. Mixing them in one task is a config error
  with a not-always-obvious message — pick one per task and
  decompose with `dependencies` if you need both.
- **Duckscript is its own dialect.** Useful because it ships in
  the binary and runs everywhere, but its syntax is neither
  bash nor PowerShell nor Lua — expect a short ramp-up and keep
  duckscript blocks small. For anything substantial, prefer a
  real shell script gated on platform.
- **Release cadence is bursty.** Versions 0.37.x have been stable
  for years with infrequent point releases. If a feature
  request lingers in issues, do not be surprised — the project
  is mature and conservative, not abandoned.
