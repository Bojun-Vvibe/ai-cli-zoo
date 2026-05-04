# gleam

- **Upstream:** https://github.com/gleam-lang/gleam
- **Version:** v1.16.0 (latest stable as of 2026-05-05)
- **License:** Apache-2.0 — [LICENCE](https://github.com/gleam-lang/gleam/blob/main/LICENCE)

## What it does

`gleam` is a single statically-linked Rust binary that is the entire
toolchain for the Gleam programming language — a strongly-typed
functional language with Hindley-Milner type inference, algebraic data
types, exhaustive pattern matching, and no `null` / no exceptions, that
compiles to **two targets**: BEAM bytecode (running on the Erlang
virtual machine alongside Erlang and Elixir code, with first-class actor
processes, OTP supervision trees, and hot-code loading) and JavaScript
(running in browsers, Node.js, Deno, Bun, and Cloudflare Workers, with
typed FFI back into the host JS runtime). The CLI covers every workflow
verb — `gleam new <name>` scaffolds a project with `gleam.toml` and a
`hello_world` template, `gleam build` typechecks and compiles to the
selected target, `gleam run` executes the project entrypoint, `gleam test`
runs the bundled test framework, `gleam format` is the Gleam-equivalent
of `gofmt` / `rustfmt` (a single canonical layout, no options to
configure), `gleam shell` opens an interactive REPL connected to the
compiled artefact, `gleam deps` manages the Hex package-registry
dependencies (shared with Erlang and Elixir), and `gleam publish`
ships a package to Hex with the API docs auto-generated from typed
signatures.

## Why it's interesting / niche

The existing language CLIs in the zoo are dominated by Rust / Go /
Python / TypeScript ecosystems plus a handful of shells (`nushell`,
`elvish`); the **typed-functional-on-BEAM** niche is empty (no Erlang
CLI, no Elixir CLI, no Gleam CLI), and the BEAM is uniquely good at the
workload classes — long-running stateful services, soft-real-time
coordination, fault-tolerant supervision trees, hot-upgrade in place —
that the rest of the runtime menu handles poorly. Gleam is also the
rare language whose CLI compiles the *same* source to two completely
different runtimes (BEAM and JS) without per-target build files; the
author writes one library, `gleam build --target erlang` produces an
Erlang module, `gleam build --target javascript` produces an ES module,
and the type system guarantees both versions have the same shape.

For LLM-CLI workflows, Gleam's properties — small surface area, no
implicit conversions, exhaustive pattern matching enforced by the
compiler, deterministic `gleam format` output — make it unusually
friendly to agent-generated code: the compiler refuses to accept
plausible-looking but wrong code that a less strict language would
silently run, and the formatter eliminates whole categories of
diff-noise from agent-authored PRs.
