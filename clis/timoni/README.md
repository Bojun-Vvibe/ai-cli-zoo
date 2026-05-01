# timoni

## What it does
A Kubernetes package manager built on CUE instead of Go templates. A `timoni.cue` module declares the resource graph in CUE — a typed configuration language with schema validation, constraints, and computed values — and `timoni bundle apply` reconciles a multi-module bundle against the cluster with server-side apply, ownership tracking, and prune semantics. Modules are distributed as OCI artifacts in any standard registry (Docker Hub, GHCR, Harbor, ECR), versioned by semver, signed via Cosign, and pulled with `timoni mod pull oci://...`.

## Why it's interesting
The Helm-without-text-templating answer: schema errors surface at `timoni mod build` time instead of at `kubectl apply` time, values get type-checked against the module's `#Config` schema, and the OCI distribution path means the same registry that hosts your container images also hosts your packages — no separate chart museum, no `helm repo add`. Pairs with `flux`/`argocd` (timoni bundles can be reconciled by either), with `cosign` (module signing/verification), and with `cue` itself for shared schema libraries across modules.

## Niche category
Kubernetes package manager — CUE-based, OCI-distributed

## Repo
https://github.com/stefanprodan/timoni

## Version pinned
`v0.26.0`

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
# macOS / Linux via Homebrew
brew install timoni

# Or via the official install script
curl -s https://raw.githubusercontent.com/stefanprodan/timoni/main/install.sh | bash
```

## Usage examples
```sh
# Install a module from an OCI registry into a namespace
timoni -n default apply podinfo oci://ghcr.io/stefanprodan/modules/podinfo \
  --version 6.5.4 --values values.cue

# Reconcile a multi-module bundle declared in bundle.cue
timoni bundle apply -f bundle.cue

# Build and push a local module to a registry
timoni mod push ./my-module oci://ghcr.io/me/modules/my-module \
  --version 1.0.0 --sign cosign
```
