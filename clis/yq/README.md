# yq

- **Repo:** https://github.com/mikefarah/yq
- **Version:** v4.53.2 (latest stable, 2026-04)
- **License:** MIT ([LICENSE](https://github.com/mikefarah/yq/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install yq` · `snap install yq` · `go install github.com/mikefarah/yq/v4@latest` · prebuilt binaries on the GitHub releases page · Docker `mikefarah/yq`

## What it does

`yq` is a single static Go binary that does for YAML / JSON /
XML / TOML / properties / CSV / TSV / Lua / shell-env files what
`jq` does for JSON: read on stdin or from a file, evaluate a
small expression language over the document tree, write back to
stdout in any of the supported formats. The expression language
is intentionally `jq`-shaped (`.foo.bar`, `.items[]`,
`select(...)`, `map(...)`, `with_entries(...)`, `|=` for in-place
updates) so anyone who already knows `jq` is immediately
productive, but it adds first-class YAML semantics that `jq`
cannot offer: comment preservation across edits (`yq '.x = 1'`
keeps the surrounding comments and blank lines), anchor and
alias handling (`*ref` / `&ref`), block-vs-flow style retention,
multi-document streams (`---`-separated, indexed as `documentIndex`),
and lossless round-tripping of YAML 1.2's full tag system.
Cross-format conversion is one flag — `yq -p yaml -o json file.yaml`,
`yq -p toml -o yaml Cargo.toml`, `yq -p xml -o json pom.xml` —
and the `-i` / `--inplace` mode rewrites the file with the
original style preserved, which is what makes it the standard
tool for "patch this Helm `values.yaml` / Kustomize overlay /
GitHub Actions workflow / Compose file in CI without mangling
the formatting". The v4.x line is the actively maintained Go
rewrite (the older v3 Python wrapper around `jq` is unrelated
and unmaintained).

## When to pick it / when not to

Pick `yq` whenever you script against YAML and want to keep the
file readable afterwards. Helm chart authors, Kustomize users,
GitOps repos (Argo CD / Flux), Ansible inventories, Compose /
Swarm stacks, GitHub Actions / GitLab CI pipelines, and
Kubernetes manifest libraries all live and die on "edit this
field, keep the rest of the file byte-stable", and `yq -i` is
the only CLI that does that reliably across every YAML feature.
The cross-format mode also makes it the easiest way to feed a
YAML file into a tool that only speaks JSON without writing a
one-off Python script.

Prefer it over [`jq`](../jq/) the moment your input is YAML or
your edit needs to round-trip comments / anchors. Prefer it over
[`dasel`](../dasel/) when you want comment preservation and a
`jq`-compatible expression language (dasel uses its own selector
syntax and does not preserve comments). Prefer it over the
Python `ruamel.yaml`-based [`kpt fn`](https://kpt.dev) /
`yamlpath` for one-shot scripting, and over the older
`kislyuk/yq` Python wrapper (which shells out to `jq` and loses
YAML structure). Skip it when you need true schema-aware editing
or graph-level refactors across many files at once — for that
reach for [`kpt`](https://kpt.dev), `kustomize edit`, or a
language-server-backed editor. Skip it for very large
(multi-GB) JSON streams where `jq --stream` is faster.

## Example

```bash
# Bump an image tag in a Helm values file in-place, keeping comments
yq -i '.image.tag = "v1.42.0"' charts/api/values.yaml

# Convert a Compose file to JSON for a tool that only speaks JSON
yq -p yaml -o json compose.yaml > compose.json

# Merge two Kubernetes manifests, the second overrides the first
yq eval-all '. as $item ireduce ({}; . * $item)' base.yaml overlay.yaml
```
