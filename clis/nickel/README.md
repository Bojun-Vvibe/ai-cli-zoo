# nickel

> **Gradually-typed configuration language with contracts and
> merging** — a single Rust binary (`nickel`) plus an LSP
> (`nls`) that evaluates `.ncl` files into JSON / YAML / TOML
> and validates them against record contracts (lightweight
> structural types checked at evaluation time, with precise
> blame on failure). Pinned to **v1.16.0** (released 2026-02-26,
> [LICENSE](https://github.com/nickel-lang/nickel/blob/master/LICENSE),
> MIT).

Source: <https://github.com/nickel-lang/nickel>

## TL;DR

`nickel` is the answer when YAML / JSON / TOML run out of
expressive power — you have ten Kubernetes manifests that
share 80 % of a base, three Terraform tfvars files that need
`if env == "prod"` branching, a Helm-values tree that wants
real merging instead of `mergeKeys: ...` hacks. A `.ncl` file
is a Nix-flavoured purely-functional record language: lazy
evaluation, first-class records with deep recursive merging
(`base & override`), type annotations (`name | String`) and
contracts (`port | std.number.PortNumber`) checked at the
boundary, list comprehensions, pattern matching, an `import`
system that resolves relative paths, and a stdlib that ships
with `std.string`, `std.array`, `std.record`, and helpers for
JSON-pointer-style overrides. `nickel export -f yaml config.ncl`
emits the rendered YAML; `nickel eval --field deploy.image`
prints one value; `nls` provides go-to-definition, hover types,
and inline diagnostics in any LSP-aware editor. The
language's design heritage is Nix — it is, in effect, "Nix the
config language without the package manager," and was
explicitly built to be reusable outside the Nix ecosystem.

## Install

```bash
# Homebrew (macOS / Linux)
brew install nickel

# Cargo (any platform with a Rust toolchain)
cargo install nickel-lang-cli nickel-lang-lsp

# Nix (with flakes enabled)
nix profile install github:nickel-lang/nickel

# Direct binary download (Linux / macOS / Windows)
curl -L -o nickel https://github.com/nickel-lang/nickel/releases/download/1.16.0/nickel-x86_64-linux
chmod +x nickel && sudo mv nickel /usr/local/bin/

# Docker image
docker pull ghcr.io/nickel-lang/nickel:1.16.0
```

## When to choose nickel

- Your config tree has cross-cutting concerns (env-aware
  values, shared base + per-target overrides, computed fields
  derived from other fields) and you keep reaching for `!!yaml`
  anchors, Helm `tpl` strings, or templating layers like
  `gomplate` / `jinja2-cli`.
- You want types and validation on config — port-number
  ranges, enum-of-strings, "this field is required if that
  field is set" — without hand-rolling JSON Schema and a
  separate validator.
- The same config feeds multiple downstream tools (Kubernetes
  + Terraform + a custom CLI) and you want one source-of-truth
  rendered to JSON / YAML / TOML per consumer.
- You like the Nix language model but do not want to adopt
  Nix the package manager.

## When to pick something else

- The config is genuinely flat and small — plain YAML / TOML
  with a JSON Schema is lighter weight and every editor knows
  it.
- The team has standardised on a different config DSL — pick
  [`cue`](https://cuelang.org/) for stronger types and a
  unification-based merge model, [`dhall`](https://dhall-lang.org/)
  for total functions and proven termination,
  [`jsonnet`](https://jsonnet.org/) for the Google/K8s ecosystem.
  All four occupy the same niche; the best one is the one your
  team will actually maintain.
- You need package management or build orchestration alongside
  the language — that is Nix's job, not Nickel's.
- You need to evaluate untrusted Nickel from a server without
  resource limits — the runtime is purely functional but is
  not currently sandboxed against compute / memory exhaustion;
  bound it externally.

## Caveats

- The language is past 1.0 and stable, but the ecosystem of
  third-party libraries is still small compared with Jsonnet
  / CUE — most "library" code lives inline in projects.
- Editor integration is via `nls`; the binary is shipped from
  the same release as `nickel` (`nls-x86_64-linux`,
  `nls-arm64-macos`, etc.) and registered as an LSP server in
  your editor against `*.ncl` files.
- `import "file.ncl"` uses lexical scope — the imported value
  cannot reference identifiers from the importer; pass them
  in explicitly via record merging or function arguments.
