# go-jsonnet

> **Pure-Go implementation of the Jsonnet config language with
> `jsonnet`, `jsonnetfmt`, and `jsonnet-lint` binaries.**
> Evaluates `.jsonnet` files into JSON, YAML, or a multi-file output
> tree — useful when you want a real programming language (variables,
> functions, imports, conditionals, list/object comprehensions) for
> generating large families of YAML/JSON config without going all the
> way to Helm or a templating language. Pinned to **v0.22.0** (released
> 2026-03-24, SPDX: `Apache-2.0`,
> [LICENSE](https://github.com/google/go-jsonnet/blob/master/LICENSE)).

Source: <https://github.com/google/go-jsonnet>

## TL;DR

`jsonnet config.jsonnet` reads a Jsonnet program and prints JSON.
Add `-y` for YAML, `-m out/` for multi-file output (one file per
top-level key — perfect for "render N Kubernetes manifests from one
program"), `-S` for raw string output (useful when the result is
itself a config snippet, not JSON). The companion `jsonnetfmt`
canonicalises formatting; `jsonnet-lint` flags unused locals,
unreachable branches, and obvious typos before evaluation.

## Install

```sh
# macOS / Linux via Homebrew
brew install jsonnet                       # installs the C++ build
brew install go-jsonnet                    # this Go implementation

# Pre-built binaries
# https://github.com/google/go-jsonnet/releases/tag/v0.22.0

# Build from source
go install github.com/google/go-jsonnet/cmd/jsonnet@v0.22.0
go install github.com/google/go-jsonnet/cmd/jsonnetfmt@v0.22.0
go install github.com/google/go-jsonnet/cmd/jsonnet-lint@v0.22.0

# Verify
jsonnet --version                          # Jsonnet commandline interpreter (Go implementation) v0.22.0
```

## License

Apache-2.0 — unrestricted commercial use, explicit patent grant.
Safe to vendor into proprietary build pipelines and CI images.

## Primary use case

You have ~200 Kubernetes manifests across 4 environments × 5
clusters that share 90% of their structure. Hand-templating with
Helm starts to fight you (Go templates inside YAML inside Helm
inside Kustomize layers); a real language is the right tool. A
~500-line Jsonnet program with `local`s, `import`s, and a
`function(env, cluster)` factory generates the entire fleet
deterministically. `jsonnet -m manifests/ -y main.jsonnet` writes
each top-level object out as its own YAML file, ready to feed into
`kubectl apply`, [`kustomize`](../kustomize/),
[`kluctl`](../kluctl/), or [`flux`](../flux/).

Other common shapes:

- **Grafana dashboard generation** — the Grafana team ships
  `grafonnet`, a Jsonnet library that turns a few hundred lines of
  programmatic dashboard description into thousands of lines of
  Grafana JSON. Maintaining the dashboards in Jsonnet is the only
  way to keep them DRY at scale.
- **Tanka / kube-libsonnet** — full Kubernetes deployments
  expressed in Jsonnet libraries; `tk apply` evaluates and applies.
- **Per-tenant config fan-out** — one input file per tenant + a
  shared Jsonnet program that produces a fully customised config
  tree per tenant in one evaluation pass.

## What it competes with

- **[`cue`](../cue/)** — also a config language, but with a
  *type-and-constraint* system instead of Jsonnet's
  *function-and-import* system. CUE proves your config is
  consistent before emitting; Jsonnet proves nothing but composes
  more freely. Pick CUE when correctness is the primary fear; pick
  Jsonnet when expressiveness and "I just want functions and
  imports over JSON" is the primary fear.
- **[`dhall`](../dhall/)** — total functional config language with
  strong types and no Turing-completeness. Even more
  correctness-leaning than CUE; smaller ecosystem. Pick when you
  want a *guarantee* that evaluation terminates and is type-safe.
- **[`nickel`](../nickel/)** — newer config language from Tweag
  with a contract system. Direct intellectual descendant of Nix +
  Jsonnet. Pick when you want gradual typing and contracts without
  going full Dhall.
- **[`pkl`](../pkl/)** — Apple's config language with a real type
  system, IDE support, and JVM/native binaries. Pick when team
  IDE/tooling polish and OO-style modelling matter more than the
  near-zero install footprint of Jsonnet.
- **`helm` / `kustomize`** — domain-specific to Kubernetes and
  template-/overlay-shaped. Jsonnet is general-purpose and produces
  raw manifests; the workflow is "Jsonnet → YAML → kubectl apply"
  and you skip Helm's templating pain.
- **Plain YAML + a Python script that prints it** — works at small
  scale; falls apart on imports, sharing, type-of-thing checks,
  and reproducibility once two people edit it.

## AI-native angle

- **Deterministic, side-effect-free evaluation.** Jsonnet has no
  I/O, no time, no random — `jsonnet x.jsonnet` is a pure
  function. That makes it ideal for LLM agent loops: "edit the
  Jsonnet, re-evaluate, diff the resulting YAML against
  expectation." The agent can verify its own change without
  needing a sandbox or a network.
- **Compact diff surface.** Editing one Jsonnet `local` can change
  hundreds of generated lines coherently — perfect for prompts of
  the form "rename this field across all environments." The agent
  edits one definition; `jsonnet -m` re-emits the whole tree;
  reviewers see the *generated* diff but the *intent* lives in one
  place.
- **`jsonnet-lint` is a cheap pre-check.** Before sending a diff
  to review, an agent can run `jsonnet-lint` on the changed file
  and surface unused-binding / unreachable-branch warnings — a
  zero-cost first-pass review that catches a class of LLM-typical
  mistakes (introducing a `local` it never uses, leaving a
  `if false then ... else ...` after refactoring).
- **No plugin model — that's the feature.** Unlike Helm or
  Kustomize, Jsonnet has no template-engine plugins, no go-template
  helpers, no patch overlays. The entire program is in the
  language. An LLM that knows Jsonnet semantics can reason about
  the *whole* program without out-of-band knowledge.
- **Library import is hermetic.** `import "lib.libsonnet"` resolves
  via `--jpath` only; there is no implicit network fetch. Agents
  cannot accidentally pull in untrusted code at evaluation time.

## Why the Go implementation specifically

The original `jsonnet` is a C++ implementation by Google. The Go
port (`go-jsonnet`) is API-compatible but ships as a single static
binary with no libstdc++ runtime dependency, which makes it the
saner choice for:

- CI runner images (Alpine, distroless) — no `libstdc++` to chase.
- Embedding via Go module — `import "github.com/google/go-jsonnet"`
  inside another tool (operators, controllers, custom CLIs).
- Cross-compilation — `GOOS=linux GOARCH=arm64 go build` Just
  Works.

Behaviour matches the C++ reference; performance is comparable for
typical config workloads. Pick `go-jsonnet` unless you have a
specific reason to want the C++ build.

## Caveats

- **Lazy evaluation surprises.** Jsonnet is lazily evaluated, so a
  `local x = expensive();` that's never referenced costs zero —
  but a typo in `expensive` won't fail until something forces it.
  Use `jsonnet-lint` to catch never-referenced bindings; use
  `std.assertEqual` to force evaluation of invariants you care
  about.
- **Error messages can be terse.** `RUNTIME ERROR: field does not
  exist: foo` with a stack trace pointing at line 412 of a
  generated `.libsonnet` is the typical bad day. Pin imports to
  versions, write thin wrappers around third-party libraries, and
  add `assert` statements at composition seams to localise blame.
- **"Manifests as code" is a discipline.** Once your Kubernetes
  fleet lives in Jsonnet, the cluster's source of truth is the
  Jsonnet program — not the YAML, not what's running. Out-of-band
  `kubectl edit` becomes drift you will discover only at the next
  full re-render. Pair with [`flux`](../flux/) /
  [`argocd`](../argocd/) to close the loop.
- **No type system.** Jsonnet is dynamically typed. Large programs
  benefit from a convention of `assert std.objectHas(x, "field");`
  guards at function boundaries — or moving to [`cue`](../cue/) /
  [`pkl`](../pkl/) if type safety is becoming the dominant pain.
- **The C++ and Go implementations occasionally drift on edge
  cases.** Stick to one in any given repo. The Go one is the
  pragmatic default in 2026.

## Concrete example

`prod.jsonnet`:

```jsonnet
local k = import "kube.libsonnet";

local app(name, replicas, image) = {
  apiVersion: "apps/v1",
  kind: "Deployment",
  metadata: { name: name, namespace: "prod" },
  spec: {
    replicas: replicas,
    selector: { matchLabels: { app: name } },
    template: {
      metadata: { labels: { app: name } },
      spec: {
        containers: [
          {
            name: name,
            image: image,
            resources: k.resources("100m", "256Mi", "500m", "512Mi"),
          },
        ],
      },
    },
  },
};

{
  api: app("api", 3, "registry.example.com/api:1.42.0"),
  worker: app("worker", 2, "registry.example.com/worker:1.42.0"),
  scheduler: app("scheduler", 1, "registry.example.com/scheduler:1.42.0"),
}
```

Render the whole tree:

```sh
jsonnet -y -m manifests/ prod.jsonnet
# manifests/api.yaml, manifests/worker.yaml, manifests/scheduler.yaml
kubectl apply -f manifests/
```

Edit `replicas: 3` to `replicas: 5` in the `app` factory and rerun
— every deployment scales coherently in one diff.
