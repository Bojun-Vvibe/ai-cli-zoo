# kubeconform

> **Fast, offline Kubernetes manifest schema validator.** A
> single Go binary that validates YAML / JSON manifests against
> the Kubernetes OpenAPI schemas (and any CRD schemas you point
> it at), without contacting an API server. Pinned to **v0.7.0**
> (per upstream releases page, verified 2026-05-01)
> ([LICENSE](https://github.com/yannh/kubeconform/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/yannh/kubeconform>

## TL;DR

`kubeconform <files…>` walks every YAML doc in the inputs and
checks each `apiVersion` + `kind` against the matching upstream
schema (cached locally after the first fetch — or fully offline
when you bundle the schema directory). It is the spiritual
successor to the abandoned `kubeval`: same UX, but parallel by
default (`-n` workers), CRD-aware via `-schema-location`
templates that resolve `{{.Group}}` / `{{.Kind}}` against your
own schema mirror, and fast enough (hundreds of files in <1 s)
to drop into a pre-commit hook or a PR check without the
"validation step takes 40 seconds" tax.

## Install

```bash
# Homebrew (macOS / Linux)
brew install kubeconform

# Go install
go install github.com/yannh/kubeconform/cmd/kubeconform@latest

# Docker
docker run --rm -v "$PWD":/work \
  ghcr.io/yannh/kubeconform:v0.7.0 -summary /work/manifests
```

## Example

```bash
# Validate a directory tree against the bundled k8s 1.30 schemas
kubeconform -strict -summary -kubernetes-version 1.30.0 manifests/

# Add CRD schemas (e.g. from datreeio/CRDs-catalog)
kubeconform \
  -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  -summary -output json \
  manifests/

# Helm chart gate: render then validate
helm template my-release ./chart | kubeconform -strict -summary -
```

## When to use

- You want a CI gate that catches typo'd `apiVersion`s,
  misspelled fields, and missing required keys *before* the
  manifest reaches the cluster.
- Your repo has CRDs (Argo, Flux, Istio, Crossplane…) and you
  need the validator to honour those schemas, not just core k8s.
- You need the check to be fast and offline (sealed-network CI,
  pre-commit hook, air-gapped build).

## When NOT to use

- You want *policy* checks ("no `:latest` tag", "must have
  resource limits", "must have a network policy") — that is
  `kyverno` / `polaris` / `kube-linter`, not schema validation.
- You want a full security posture / CIS scan — reach for
  `kubescape` or `trivy config`.
- You need server-side admission semantics (defaulting,
  conversion webhooks) — only a real API server gives that;
  `kubeconform` checks the static schema, not the live cluster.
