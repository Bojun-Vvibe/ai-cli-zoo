# dhall

- **Repo:** https://github.com/dhall-lang/dhall-haskell
- **Version:** 1.42.2 (latest stable, January 2025)
- **License:** BSD-3-Clause ([LICENSE](https://github.com/dhall-lang/dhall-haskell/blob/main/LICENSE))
- **Language:** Haskell (reference implementation; alternative implementations in Rust, Go, Ruby, Clojure, Python)
- **Install:** `brew install dhall dhall-json dhall-yaml` · download static binaries from the GitHub release page · binary names are `dhall`, `dhall-to-json`, `dhall-to-yaml`, `json-to-dhall`, `yaml-to-dhall`

## What it does

Dhall is a **typed, total, hashable, importable** configuration
language whose CLI compiles `.dhall` programs down to JSON, YAML,
TOML, XML, plain text, or Bash variables. Unlike YAML it has a
real type system: `{ port : Natural, host : Text, tls : Bool }` is
checked at compile time, so a typo in a key, a string where a number
belongs, or a missing required field becomes a compile error before
the config ever reaches your service. Unlike Helm-style template
languages it is **total** — no `while`, no untyped `range`, every
program either evaluates to a value or fails at type-check; you
cannot ship a config that loops or crashes the renderer at runtime.

The "importable" part is the load-bearing wall. A Dhall expression
can `let utils = ./utils.dhall` (relative file), `let prelude =
https://prelude.dhall-lang.org/package.dhall sha256:…` (URL with
mandatory hash pin), or `let secret = env:DATABASE_URL` (env var)
— and the CLI's `dhall freeze` walks every import and pins the
SHA-256, so once frozen, an upstream URL change cannot silently
mutate your config. `dhall hash` prints the integrity hash of any
expression; `dhall lint` removes shadowed bindings and dead code;
`dhall format` is `gofmt` for `.dhall` files. The companion
binaries (`dhall-to-yaml`, `dhall-to-json`, etc.) make Dhall a
*generator* for whatever format your downstream tool already speaks.

## When to pick it / when not to

Pick Dhall when you have **YAML or JSON config that has grown teeth**
— Kubernetes manifests across N environments, GitHub Actions
workflows that copy-paste 80% of the same job between repos, large
CI matrices, Terraform `tfvars` files diverging across stages,
service configs where a typo means a 3 a.m. page. Writing a typed
`Service` record, a `prod : Service`, a `staging : Service` that
inherits from `prod` with overrides, and emitting all of them via
`dhall-to-yaml` collapses both the duplication and the typo class.

Pick it when you want **strong supply-chain guarantees** for shared
configuration: SHA-pinned imports + offline `dhall freeze` give you
"this YAML file is the deterministic output of this exact program
graph", which is the property a generated Helm `values.yaml` or a
generated `kustomization.yaml` should have but rarely does.

Skip it when:

- Your config is **small and stable** — a 30-line `application.yaml`
  doesn't pay back the cost of teaching the team a new language.
- You need **runtime templating against live data** (e.g., values
  fetched from Vault at deploy time) — Dhall is a pure compile-time
  language; mix it with [`sops`](https://github.com/getsops/sops)
  or [`vault agent template`](https://developer.hashicorp.com/vault)
  for the secret-injection step.
- The downstream tool **requires its own templating** ([Helm](https://helm.sh/)
  charts shared publicly) — emit a vanilla manifest from Dhall and
  hand it to `kubectl apply` directly, or accept that you're
  swapping Helm's templating for Dhall's and pick one.
- You want a **mainstream-IDE-friendly DSL** — Dhall has VS Code,
  Vim, and Emacs LSPs, but it's still a niche language; for "JSON
  with comments and types"-shaped problems, [CUE](https://cuelang.org/)
  has a larger community and overlapping feature set.

## Why it matters in an AI-native workflow

LLM agents asked to "edit this config" are catastrophically bad at
inferring the schema of a YAML file. They invent keys, drop
required fields, swap `int` for `string`, and produce diffs that
"look right" but break at deploy. Dhall's compile-time types fix
this at the source: the agent edits a typed `.dhall` program and
runs `dhall type ./config.dhall` before committing — a wrong key or
wrong type aborts immediately with a precise error pointing at the
offending line, instead of silently breaking production.

The frozen-import model is the second leverage point. An agent
fetching a shared schema from a URL with `dhall freeze` cannot be
poisoned by a later upstream change; the SHA pin makes the
context the agent reasoned over identical to the artifact CI will
type-check. And because Dhall is total, an agent's "let me just
generate the YAML and inspect it" loop is bounded — `dhall-to-yaml`
either prints the value or fails at type-check, never spins or
emits half a file.

## Example invocations

```bash
# Compile a Dhall program to YAML.
dhall-to-yaml --file ./k8s/deployment.dhall > deployment.yaml

# Compile to JSON, preserving null-handling.
dhall-to-json --file ./config.dhall > config.json

# Type-check without rendering output.
dhall type --file ./config.dhall

# Format in place (gofmt-style).
dhall format --inplace ./config.dhall

# Lint — remove shadowed bindings, simplify.
dhall lint --inplace ./config.dhall

# Freeze every URL import to a SHA-256 hash, pinning the supply chain.
dhall freeze --inplace ./config.dhall

# Print the SHA-256 hash of an expression (for cache keys, integrity checks).
dhall hash --file ./config.dhall

# Diff two Dhall expressions semantically (alpha-equivalent — not text diff).
dhall diff ./old.dhall ./new.dhall

# Convert existing YAML back to a Dhall starting point for refactoring.
yaml-to-dhall './K8s/Deployment.dhall' --file ./deployment.yaml > deployment.dhall

# Render a Bash variable file from a typed config.
dhall-to-bash --declare PORT --file <(echo '8080')
```

## Caveats

- The default Haskell binary has a non-trivial startup cost (~200 ms
  cold). For tight loops, the Rust implementation
  ([`dhall-rust`](https://github.com/Nadrieril/dhall-rust)) and the
  Go one ([`dhall-golang`](https://github.com/philandstuff/dhall-golang))
  are faster but lag the spec slightly — verify feature parity for
  what you use.
- URL imports without a SHA hash will fetch on every evaluation and
  trust the upstream. **Always run `dhall freeze` before committing**
  imports of shared schemas; treat unfrozen URL imports the way you
  would treat unpinned `npm install` arguments.
- Dhall's `Natural` is unbounded; the JSON / YAML it emits will use
  bare integers, which can overflow downstream consumers expecting
  64-bit signed. If a value can plausibly exceed `2^63-1`, render
  it as text and let the consumer parse.
- The language is small but not trivial. Expect a one-week ramp for
  a developer comfortable with strongly-typed FP, longer for one
  coming purely from JSON / YAML. The
  [Prelude](https://prelude.dhall-lang.org/) has most of the
  building blocks you'll need; reach for it before writing your own.
- Dhall has *no* I/O at evaluation time other than imports and
  environment variables. It cannot read a database, call an HTTP
  API, or shell out — that's by design (totality), but it means
  "fetch this value at deploy time" needs a separate post-processing
  step.
