# dasel

- **Repo:** https://github.com/TomWright/dasel
- **Version:** v2.8.1 (latest stable, 2025)
- **License:** MIT ([LICENSE](https://github.com/TomWright/dasel/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install dasel` · `go install github.com/tomwright/dasel/v2/cmd/dasel@master` · binary releases on the GitHub release page · also packaged in nixpkgs, scoop, and snap

## What it does

`dasel` (data selector) is a single binary that reads, queries, and edits
JSON, YAML, TOML, XML, and CSV using one selector syntax across all of them.
The pitch is structural: instead of memorising `jq` for JSON, `yq` for YAML,
`tomlq` for TOML, and a separate XPath helper for XML, you learn one path
language (`.foo.bar.[0].name`, `.users.(name=alice)`, `.items.all().price`)
and pipe any of the five formats through the same command. Conversion is a
side effect of selection: `dasel -r json -w yaml '.' < in.json > out.yaml`
turns a JSON document into YAML, with comments preserved on YAML→YAML edits
and key order preserved within the round-trip the format supports.
In-place edits (`dasel put`, `dasel delete`) update structured config files
without templating: `dasel put -r yaml -t string -v 1.21 -f go.mod.yaml
'.toolchain'` rewrites just that key, leaves the rest of the file byte-identical
to the extent the format allows, and exits non-zero if the path is missing
unless you pass `-c` (create). Selectors compose: `.images.all()
.tags.append()` adds a tag to every image, `.users.filter(.role=admin)`
narrows a list, `.metrics.sum(.bytes)` reduces. Output formats are independent
of input formats, so `dasel -r yaml -w json '.' k8s.yaml | jq` is a perfectly
fine bridge into the rest of a JSON-shaped pipeline.

## When to pick it / when not to

Pick `dasel` when a workflow touches **more than one structured format** —
typically Helm chart values (YAML) referencing image tags written into a
JSON manifest, or a Cargo.toml + GitHub Actions YAML pair you want to bump
in lockstep. One binary, one selector grammar, no per-format flag dance.
It is also the path of least resistance for **format conversion in shell
pipelines**: `aws ssm get-parameter ... --output json | dasel -r json -w
yaml '.Parameter.Value' | kubectl apply -f -` is a real shape that drops
in without a Python detour. Reach for it in CI bumpers ("update the image
tag in 14 YAML files"), in scaffolding scripts ("read this TOML, write a
JSON for the next tool"), and as the all-purpose `--in-place` editor for
config files where you do not want to ship `sed` regex against indentation.
Pair with [`yq`](../yq/) (when you specifically want jq syntax on YAML),
with [`jaq`](../jaq/) (faster pure-Rust jq for JSON-only pipelines), and
with [`csvlens`](../csvlens/) for previewing the CSV branch.

Skip it when you only ever touch JSON and you already know `jq` — `jq` /
`jaq` have a richer expression language (variables, recursive descent,
streaming, modules, custom functions) than dasel's selector. Skip it for
**round-trip-perfect YAML edits** where every comment, anchor, and quoting
style must survive — dasel preserves comments on simple key edits but is
not a YAML AST editor; for that go to `yq` (Mike Farah, Go) which uses
go-yaml v3 with comment threading, or to a real round-trip library
(`ruamel.yaml`). Skip it for **schema-validating reads** — dasel does not
know your schema; combine with `jsonschema`, `cue vet`, or `kubeconform`
for that. Skip it for **enormous JSON streams** that need lazy / streaming
evaluation — `jq --stream` exists, dasel loads the document. And skip it
for HCL, Cue, Pkl, or Dhall — dasel covers JSON / YAML / TOML / XML / CSV;
the configuration-language formats need their own toolchain.

## Example invocations

```bash
# Read a single value from JSON
dasel -r json '.user.name' < user.json

# Same selector, different format
dasel -r yaml '.user.name' < user.yaml

# Convert YAML to JSON in a pipeline
dasel -r yaml -w json '.' < deploy.yaml | jq

# In-place edit: bump an image tag in a Helm values file
dasel put -r yaml -t string -v 'v1.4.2' -f values.yaml '.image.tag'

# Filter a list by field
dasel -r json '.users.filter(.role=admin)' < users.json

# Append to an array
dasel put -r json -t string -v 'newtag' -f manifest.json '.tags.append()'

# Map a function across a list
dasel -r json '.items.all().price' < cart.json

# Delete a key
dasel delete -r yaml -f config.yaml '.deprecated_section'

# Cross-format sync: read TOML version, write into a JSON manifest
VER=$(dasel -r toml '.package.version' < Cargo.toml)
dasel put -r json -t string -v "$VER" -f package.json '.version'

# Validate that a path exists (exit non-zero if missing)
dasel -r yaml '.spec.template.spec.containers.[0].image' < pod.yaml > /dev/null

# CSV: pull a column as JSON array
dasel -r csv -w json '.[*].email' < contacts.csv
```
