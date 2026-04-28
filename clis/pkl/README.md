# pkl

- **Repo:** https://github.com/apple/pkl
- **Version:** 0.31.1 (2026-03-26)
- **License:** Apache-2.0 ([LICENSE.txt](https://github.com/apple/pkl/blob/main/LICENSE.txt))
- **Language:** Java / Kotlin (CLI ships as a GraalVM native-image binary)
- **Install:** `brew install pkl` · `curl -L -o pkl https://github.com/apple/pkl/releases/download/0.31.1/pkl-macos-aarch64 && chmod +x pkl` · prebuilt binaries on the GitHub release page · binary name is `pkl`

## What it does

`pkl` ("Pickle") is a **typed configuration language** and
companion CLI from Apple. You write `.pkl` files in a syntax that
looks like a hybrid of TypeScript and HOCON — typed records, late
binding, modules with imports, lambdas, list / map comprehensions,
amends-based inheritance — and the CLI evaluates them down to
**JSON, YAML, plist, XML, Java properties, or `.pcf`** for
consumption by anything that reads structured config. The type
checker catches the entire class of "I forgot a comma in
production.yaml at 3am" bugs at evaluation time, before the
config ever reaches the service that has to load it.

The CLI is one static binary (`pkl`) plus a set of optional
codegen tools (`pkl-codegen-java`, `pkl-codegen-kotlin`, `pkl-gen`
for Go / Swift / TypeScript via plugins) that emit native data
classes from a `.pkl` schema, so the Pkl module becomes the single
source of truth and the service code never re-parses YAML by
hand. `pkl eval`, `pkl repl`, and `pkl test` (a test runner for
config) round out the toolchain. A long-running `pkl server` mode
speaks the Pkl language-server protocol for editor integration.

## When to pick it / when not to

Reach for `pkl` when **your YAML / JSON config has grown a type
system in your head that the file format cannot express**:
Kubernetes manifests with cross-references, multi-environment
service config that should share 90 % of its values, feature-flag
matrices, infrastructure templates that are currently a Helm chart
held together with `tpl`. Pkl gives you real types, real imports,
real functions, and emits the boring YAML at the end so consumers
do not need to learn anything new.

Skip it for a five-line config (`.env` is fine). Skip when your
team is already deeply invested in Jsonnet / CUE / Dhall and the
ecosystem cost of switching outweighs the benefit. Skip when the
config consumer is itself a Pkl-unaware tool that you cannot
re-render through `pkl eval` in CI — Pkl is most valuable when
the build pipeline owns the eval step.

## Why it matters in an AI-native workflow

LLM-generated config is notoriously brittle: the model emits YAML
that *looks* right but mis-indents a key, drops a required field,
or hallucinates a list shape that the consumer rejects at runtime.
Authoring config in Pkl shifts that failure mode left — the model
emits `.pkl`, the type checker rejects nonsense before the file is
ever rendered, and the agent gets a structured error message it
can act on. Schemas authored in Pkl also double as **tool
input schemas** for MCP servers and as the source for codegen
into the agent's own dataclasses, so the same artifact governs
"what config is valid" and "what shape does the agent return".

## Example invocations

```bash
# Evaluate a Pkl module and emit YAML
pkl eval -f yaml config.pkl

# Emit JSON to stdout
pkl eval -f json config.pkl

# Render a multi-file output (e.g. one Kubernetes manifest per service)
pkl eval -m ./out -f yaml services.pkl

# Run config tests (assertEquals / assertThat in .pkl files)
pkl test tests/

# Open the REPL to poke at types interactively
pkl repl

# Generate Java data classes from a Pkl schema
pkl-codegen-java --output-dir src/main/java schema.pkl

# A trivial config.pkl looks like:
#   amends "package://pkg.pkl-lang.org/pkl-k8s/[email protected]#/api/apps/v1/Deployment.pkl"
#   metadata { name = "agent-runner" }
#   spec { replicas = if (System.env["ENV"] == "prod") 5 else 1 }
```

## Alternatives in this catalog

- [`dasel`](../dasel/) — query / convert between JSON / YAML / TOML
  / XML; reach for dasel when the config already exists and you
  need to reshape it, pkl when you are *authoring* it.
- [`yq`](../yq/) / [`jq`](../jq/) — sibling story for one-off
  YAML / JSON edits in a pipeline.
- [`kustomize`](../kustomize/) — Kubernetes-specific overlay /
  patch system; pick kustomize when you only need K8s and want
  zero new languages, pkl when you want a real type system that
  also covers non-K8s config.
- [`helm`](../helm/) — Go-template-based K8s charts; pkl is the
  "modern typed config" answer to helm's "everything is a string
  template" problem.
