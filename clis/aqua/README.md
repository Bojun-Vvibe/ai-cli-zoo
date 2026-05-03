# aqua

> **Declarative CLI version manager: one `aqua.yaml` checked into the
> repo pins every developer-tool binary (kubectl, terraform, gh, jq,
> helm, golangci-lint, …) to an exact version + SHA-256 from a
> community-curated registry, with lazy on-demand install via shim
> binaries on `PATH`.** Pinned to **v2.57.2**, MIT
> ([LICENSE](https://github.com/aquaproj/aqua/blob/main/LICENSE)).

- **Repo:** https://github.com/aquaproj/aqua
- **Latest version:** v2.57.2 (2026-04-25)
- **License:** MIT (`LICENSE` at repo root)
- **Category:** `tool-manager` / `version-pinning` / `supply-chain` / `dev-environment`
- **Language:** Go

## What it does

`aqua` is a polyglot CLI version manager whose source of truth is a
plain `aqua.yaml` file you commit to the repo. Each entry pins a
package to an exact version (`packages: [{ name: kubernetes/kubectl@v1.31.2 }]`),
and the resolver pulls the binary from the corresponding GitHub
release (or other configured source) into a content-addressed cache
under `~/.local/share/aquaproj-aqua/`. The novel piece is the
*shim*: instead of mutating `PATH` to point at the active version
the way `nvm` / `pyenv` / `rbenv` do, `aqua` writes a tiny shim
binary named after every managed tool into one PATH directory
(`~/.local/share/aquaproj-aqua/bin`), and on first invocation the
shim reads the nearest `aqua.yaml` walking up from `$PWD`, downloads
the pinned version if absent, verifies SHA-256 + (optionally)
Cosign signature + (optionally) SLSA provenance, and exec's the
real binary. The result is per-directory tool versions with no
shell-hook overhead and no `aqua use` step — `cd ~/repos/foo && kubectl version`
runs the kubectl version foo's `aqua.yaml` declares; `cd ~/repos/bar`
runs a different one. The community **standard registry**
(`aquaproj/aqua-registry`) ships ~1,800 curated package definitions
with checksum URLs and version-overrides, so adding a new tool is
typically `aqua g -i kubernetes/kubectl` (interactive picker) or
`aqua g kubernetes/kubectl@v1.31.2` (deterministic), not
hand-writing a manifest.

## Install

```bash
# macOS / Linux via Homebrew
brew install aquaproj/aqua/aqua

# All platforms — install script (verifies its own checksum)
curl -sSfL https://raw.githubusercontent.com/aquaproj/aqua-installer/v3.0.1/aqua-installer | bash

# After install: add the shim dir to PATH
export PATH="${AQUA_ROOT_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/aquaproj-aqua}/bin:$PATH"
```

## Examples

```bash
# Initialise an aqua.yaml in the current repo
aqua init

# Add a tool, pinning to a specific upstream release
aqua g -i kubernetes/kubectl    # interactive version picker
aqua g hashicorp/terraform@v1.10.0

# Install everything aqua.yaml declares (no global state mutated)
aqua i

# Update the lockfile after bumping versions
aqua update

# Verify checksums + Cosign signatures + SLSA provenance on install
# (configured per-package in the registry; opt-in policy enforcement)
aqua policy allow             # trust the policy file once
aqua i                        # subsequent installs verify

# Generate a per-package policy file for review-then-allow workflow
aqua policy init
```

## Why it matters in an AI-native workflow

Agent-driven dev workflows multiply the number of CLIs in play —
one PR-triage agent wants `gh` + `jq` + `gron`, the next wants
`kubectl` + `helm` + `kubeconform`, the third wants `terraform` +
`tflint` + `infracost`. The cross-machine drift problem
(`kubectl` 1.28 on the agent's laptop, 1.31 in CI, 1.30 on the
review reviewer's box) is the same problem the language-runtime
managers ([`mise`](../mise/), [`asdf`](../asdf/), [`pkgx`](../pkgx/))
already solve for `node` / `python` / `ruby`, but those tools
historically focus on language toolchains and load plugins from a
network-fetched script per language, which is itself a
supply-chain surface. `aqua`'s contribution is the
*registry-as-curated-allowlist* model: the package definition,
checksum URL, signature key, and version-override rules live in
one git repo (`aquaproj/aqua-registry`) reviewed by maintainers,
and the local `aqua.yaml` is a small set of pins against that
registry. Pairs with [`mise`](../mise/) (mise for language
runtimes — node / python / go / rust toolchains; aqua for the
~1,800 binary CLIs sitting on top), [`pkgx`](../pkgx/) (pkgx is
invocation-scoped, no committed manifest; aqua is repo-scoped,
manifest is the source of truth), [`asdf`](../asdf/) (asdf is the
historical baseline with shell-hook PATH switching and a plugin
per tool; aqua replaces the plugin sprawl with one registry +
shim binaries). Complementary to [`cosign`](../cosign/) (aqua
verifies signatures aqua-side) and [`slsa-verifier`](../slsa-verifier/)
(aqua verifies SLSA provenance aqua-side), so the CI gate
"this PR's tooling matches what the maintainer actually published"
becomes a single `aqua i` call.
