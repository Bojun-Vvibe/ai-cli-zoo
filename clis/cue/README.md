# cue

## What it does
A typed configuration language and CLI that unifies schema, constraint, and data into a single expression. `cue vet data.yaml schema.cue` validates arbitrary JSON/YAML against a CUE schema; `cue export` renders CUE down to JSON/YAML/TOML; `cue eval` resolves cross-file constraints, defaults, and computed values. The language is a superset of JSON with types, disjunctions, optional fields, regex constraints, and a powerful unification operator that merges configurations from multiple sources without conflict surprises.

## Why it's interesting
The rare config language that is simultaneously a schema language: the same `#Server: { port: int & >0 & <65536, name: =~"^[a-z]+$" }` definition both validates inputs *and* generates outputs. This collapses the usual "JSON Schema for validation + Helm templates for generation + YAML anchors for reuse" stack into one tool. Powers [`timoni`](../timoni/) (CUE-based Kubernetes packages), `dagger` (CUE-described pipelines in earlier versions), and is increasingly used as the schema layer for OPA policies, Kubernetes admission, and config-as-code platforms. Unlike Jsonnet, CUE refuses to evaluate ambiguous merges — drift between teams' overrides surfaces at `cue vet` time, not in production.

## Niche category
Typed configuration language — schema + data + constraint, unified

## Repo
https://github.com/cue-lang/cue

## Version pinned
`v0.16.1`

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
# macOS / Linux via Homebrew
brew install cue

# Or via go install
go install cuelang.org/go/cmd/cue@latest
```

## Usage examples
```sh
# Validate a YAML file against a CUE schema
cue vet config.yaml schema.cue

# Export CUE down to JSON (or yaml, toml, text)
cue export config.cue --out json

# Evaluate and pretty-print with all constraints resolved
cue eval -c config.cue

# Import an existing JSON/YAML file as starter CUE
cue import config.yaml

# Run a CUE workflow / module command
cue mod init example.com/myconfig
cue mod tidy
```

## When to pick `cue` vs alternatives
- **vs Jsonnet**: Jsonnet is a templating language with imports and inheritance; CUE is a constraint language with bidirectional schema/data. Pick Jsonnet when you want familiar OOP-style config inheritance; pick CUE when you want type errors at validation time and refuse-to-merge semantics for conflicting overrides.
- **vs JSON Schema + Helm**: that combo separates "is this valid?" from "render this template," requiring two tools and two mental models. CUE does both with one definition and one CLI.
- **vs `dhall`**: Dhall is a pure functional config language with strong totality guarantees; CUE is more pragmatic and YAML-interop-first. Dhall fans value the lambda calculus foundation; CUE users value `cue import legacy.yaml` working out of the box.
- **vs HCL / Terraform**: HCL is purpose-built for Terraform and shines inside that ecosystem; CUE is general-purpose and stronger for cross-system config (one schema validating Kubernetes manifests, CI configs, and app settings simultaneously).
- **Pairs with**: [`timoni`](../timoni/) (CUE-native K8s packages), [`conftest`](../conftest/) (policy testing — Conftest uses Rego, but CUE's `vet` covers many of the same use cases for typed configs), and any GitOps flow where you want PR-time validation instead of cluster-time rejection.
