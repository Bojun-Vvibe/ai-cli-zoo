# yq

- **Repo:** https://github.com/mikefarah/yq
- **Version:** v4.53.2 (latest stable, 2026-04)
- **License:** MIT ([LICENSE](https://github.com/mikefarah/yq/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install yq` · `go install github.com/mikefarah/yq/v4@latest` · `snap install yq` · `nix run nixpkgs#yq-go` · static binary releases on the GitHub release page · `docker run --rm -v "${PWD}":/workdir mikefarah/yq`

## What it does

`yq` is a portable command-line YAML / JSON / XML / TOML / properties /
CSV / TSV / Lua / `.env` / base64 processor with a syntax that is a
deliberate near-clone of `jq`'s. The binary is a single static Go
executable (~6 MB) with no runtime dependency, which is the reason it has
become the default "shell-friendly YAML editor" in Kubernetes manifests,
GitHub Actions workflows, Helm charts, Ansible playbooks, and Docker
Compose files. The expression language supports `.foo.bar`, `.[]`,
`select()`, `map()`, `to_entries`, `has`, `length`, `keys`, `paths`,
arithmetic, regex (`test`, `match`, `capture`, `sub`), string manipulation
(`split`, `join`, `ascii_upcase`, `trim`), variable bindings (`as $x`),
`reduce`, anchors / aliases (preserved or expanded with `--explode`), and
custom functions via `def`. Crucially for YAML editing, `yq` round-trips
**comments and key order** by default — this is the property `jq | yj`
pipelines famously break, and the reason a `kubectl get -o yaml | yq` →
edit → re-apply loop produces a clean diff instead of a 400-line reorder.
The `-i` flag does in-place edits without a `>` shell-redirect footgun.
Format conversion is one flag: `yq -p json -o yaml`, `yq -p xml -o json`,
`yq -p toml -o yaml`, `yq -p csv -o json` — the same expression engine
runs across all of them. Multi-document YAML (`---` separated) is
first-class: `yq 'select(.kind == "Deployment")' all.yaml` walks every
doc in the stream. The `eval-all` (`ea`) subcommand loads multiple files
or all documents into a single expression for merges, diffs, and
cross-file references — `yq ea '. as $item ireduce ({}; . * $item)' a.yaml
b.yaml c.yaml` is the canonical "deep merge N files in dependency order"
recipe.

## When to pick it / when not to

Pick `yq` when the input is YAML and you need either (a) shell-pipeline
ergonomics (CI step, `Makefile` target, git hook, Dockerfile RUN), (b)
comment- and order-preserving edits to a hand-maintained config file, or
(c) format conversion as part of a pipeline (`yq -p yaml -o json
values.yaml | jq …`). It is the right tool for `helm template … | yq
'select(.kind == "Ingress") | .spec.rules[].host'`, for
`kubectl get cm app-config -o yaml | yq '.data.LOG_LEVEL = "debug"' |
kubectl apply -f -`, and for `yq -i '.services.api.image = "myrepo/api:" +
strenv(TAG)' docker-compose.yaml` in a release script. Pair with
[`jq`](../jq/) or [`jaq`](../jaq/) when downstream stages are JSON-only,
with [`dasel`](https://github.com/tomwright/dasel) when you want a
single binary that speaks even more formats but with a less `jq`-shaped
syntax, and with [`fx`](../fx/) when you have already converted to JSON
and want an interactive viewer.

Skip it when the input is plain JSON and the team already knows `jq` —
`yq -p json` works fine but you are paying a binary for no benefit.
Skip it for very large YAML streams where the comment-preserving parser
becomes a bottleneck (it allocates more than a stream parser would); a
purpose-built `kustomize` / `helm` flow or a `python -c "import yaml;
yaml.safe_load_all(...)"` script with `ruamel.yaml` will be faster on
multi-hundred-MB manifest dumps. Skip it when you need a *typed* schema
guarantee on the output — `yq` will happily produce YAML that fails CRD
validation; for that reach for `kubeconform`, `kustomize build | conftest`,
or language-native libraries with the schema bound. There is also a
separate **Python `yq`** by Andrey Kislyuk (`pip install yq`) that
wraps `jq`; this entry is the Go `yq` by Mike Farah, which is the one
shipped by Homebrew and the one most CI examples target — confirm the
binary with `yq --version` (it prints `mikefarah/yq` in the output).

## Example invocations

```bash
# Read a value (works on YAML or JSON the same way)
yq '.spec.replicas' deployment.yaml

# Edit in place, preserving comments and key order
yq -i '.spec.replicas = 5' deployment.yaml

# Multi-doc YAML: pick one kind from a stream
yq 'select(.kind == "Service") | .metadata.name' k8s/all.yaml

# Convert formats with the same expression engine
yq -p json -o yaml '.' package.json
yq -p xml -o json '.' pom.xml | jq '.project.dependencies'

# Deep-merge N files (later wins)
yq ea '. as $item ireduce ({}; . * $item)' base.yaml override.yaml prod.yaml

# Inject an environment variable safely (handles quoting / type coercion)
TAG=v1.4.2 yq -i '.services.api.image = "myrepo/api:" + strenv(TAG)' compose.yaml

# Bulk-rename a key across every document in a multi-doc stream
yq -i '(.. | select(has("oldKey")) | .newKey) = (.oldKey) | del(.. | .oldKey?)' manifests.yaml

# Strip null / empty fields before applying (cleaner kubectl diff)
yq 'del(.. | select(. == null or . == "" or . == [] or . == {}))' resource.yaml
```
