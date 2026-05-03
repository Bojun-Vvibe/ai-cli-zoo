# krew

- **Repo:** https://github.com/kubernetes-sigs/krew
- **Version:** v0.5.0 (2026-02-26)
- **License:** Apache-2.0 ([LICENSE](https://github.com/kubernetes-sigs/krew/blob/master/LICENSE))
- **Language:** Go
- **Install:** official bootstrap script (`(set -x; cd "$(mktemp -d)" && OS="$(uname | tr '[:upper:]' '[:lower:]')" && ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" && KREW="krew-${OS}_${ARCH}" && curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" && tar zxvf "${KREW}.tar.gz" && ./"${KREW}" install krew)`) · `brew install krew` · then add `${KREW_ROOT:-$HOME/.krew}/bin` to `PATH`

## What it does

`krew` is the package manager for `kubectl` plugins, governed under the Kubernetes SIG CLI umbrella. After `kubectl krew install <plugin>` it drops a binary named `kubectl-<plugin>` into `~/.krew/bin/`; because `kubectl` resolves any executable named `kubectl-foo` on `PATH` as the subcommand `kubectl foo`, the plugin is invokable as a first-class `kubectl` verb the moment install finishes — no shell rc edit, no per-plugin instructions. `krew` itself is just another kubectl plugin (`kubectl krew`), so you bootstrap it once with the official installer and from then on it manages itself plus everything else. The plugin index is a curated, security-reviewed Git repo at [kubernetes-sigs/krew-index](https://github.com/kubernetes-sigs/krew-index) — every plugin manifest is a YAML file with a name, version, per-OS/arch download URLs, sha256 sums, and an SPDX-style license declaration; `krew` enforces signature-style integrity (sha256 verification at install time) and a basic plugin-authoring policy (must declare a license, must not collide with an existing kubectl verb, must publish per-platform binaries). The day-to-day surface: `kubectl krew search <term>` greps the index by name and short description; `kubectl krew info <plugin>` shows the manifest detail (homepage, version, platforms, caveats); `kubectl krew install <plugin>` downloads the right per-OS/arch tarball, verifies the sha256, and unpacks the binary into `~/.krew/store/<plugin>/<version>/`; `kubectl krew upgrade` brings every installed plugin to the latest indexed version in one shot; `kubectl krew list` shows what is installed and at which versions; `kubectl krew uninstall <plugin>` removes both the binary and the store entry. The index has 200+ plugins now — `kubectl ctx`, `kubectl ns`, `kubectl tree`, `kubectl neat`, `kubectl view-secret`, `kubectl rolesum`, `kubectl whoami`, `kubectl access-matrix`, `kubectl df-pv`, etc. — which means most of the "I wish kubectl could just…" small utilities are one `krew install` away. Custom indexes (`kubectl krew index add <name> <git-url>`) let teams ship private plugin indexes alongside the public one without forking the tool.

## When to pick it / when not to

Pick `krew` whenever you have decided that `kubectl` plus 5–15 small focused plugins is your operator UX, instead of one heavy "k8s super-CLI". Concrete cases: you maintain a runbook that says "to debug pod X, run `kubectl tree`, then `kubectl neat`, then `kubectl view-secret`" — `krew install tree neat view-secret` is the one-liner that makes the runbook reproducible on every new on-call laptop and CI agent; you ship a small set of internal plugins (`kubectl-prod-shell`, `kubectl-revoke-token`) and want a uniform install path for them — point `krew` at a private index repo and `krew install company/prod-shell` works the same as the public ones; you want `kubectl rolesum` / `kubectl access-matrix` for RBAC review without pulling in a heavy GUI; you are setting up a new dev container image and want a deterministic, sha256-verified install of "the standard set of kubectl plugins". Pair with [`kubectl-neat`](../kubectl-neat/) (one of the plugins krew installs), [`kubectl-tree`](../kubectl-tree/), [`kubectx`](../kubectx/) (also krew-installable), [`stern`](../stern/) (logs across pods, also krew-installable as `kubectl stern`), and [`k9s`](../k9s/) (TUI, separate install). Use [`kdash`](../kdash/) or [`k9s`](../k9s/) for the cluster TUI side; krew is for the discrete plugin loadout.

Skip `krew` when your team has standardized on a single super-CLI like [`k9s`](../k9s/) or [`kdash`](../kdash/) and the long tail of small plugins is not how you operate; krew adds a process and a manifest-store you do not need. Skip when your environments forbid downloading binaries from arbitrary GitHub-release URLs and you have not whitelisted `github.com` releases — krew installs are sha256-verified but they are still binary downloads from the plugin's own release artifacts. Skip when the only plugin you actually want is one — `kubectl krew install foo` is fine, but if it is *literally one* and you already package binaries internally, just shipping `kubectl-foo` on `PATH` is simpler. Skip when you need supply-chain controls beyond sha256 (sigstore / cosign signature verification of every plugin) — krew enforces sha256 of the manifest-declared archive but does not currently verify cosign signatures of upstream plugin builds, so a stricter pipeline should curate a private index that pins specific plugin versions you have separately audited.

## Vs already cataloged

- **Vs [`kubectl-neat`](../kubectl-neat/), [`kubectl-tree`](../kubectl-tree/):** krew is the *manager*; those are *plugins*. The canonical install path for both is `kubectl krew install neat` and `kubectl krew install tree`. Catalog those entries for what each plugin does; this entry is for the management layer that gets them onto the box.
- **Vs [`k9s`](../k9s/) / [`kdash`](../kdash/):** different philosophy. k9s and kdash are integrated cluster TUIs that give you one rich UI for everything; krew embraces small composable `kubectl-foo` plugins. Many operators run both — k9s for live cluster navigation, krew-managed plugins for one-shot CLI tasks (`kubectl tree`, `kubectl access-matrix`).
- **Vs [`asdf`](../asdf/) / [`mise`](../mise/) / [`pkgx`](../pkgx/):** those are general-purpose multi-language version managers; krew is `kubectl`-plugin-specific. krew speaks the `kubectl` plugin model (binary named `kubectl-<verb>` on `PATH`) and the krew-index manifest schema, neither of which the general managers know about.
- **Vs [`helm`](../helm/) / [`kustomize`](../kustomize/):** orthogonal. Those manage cluster-side manifests; krew manages client-side `kubectl` extensions. They never overlap.
- **Vs [`kubectx`](../kubectx/):** kubectx is one of the most-installed krew plugins (`kubectl ctx`, `kubectl ns`). Same relationship as krew↔kubectl-neat.

## Caveats

- **`PATH` setup is on you.** The bootstrap installer drops `kubectl-krew` into `~/.krew/bin/` but does not edit your shell rc. Add `export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"` to `~/.bashrc` / `~/.zshrc` / `~/.config/fish/config.fish`. Skipping this is the #1 "I just installed krew and `kubectl krew` says command not found" issue.
- **Plugin trust is index-level.** The default index is curated by the SIG, but each plugin's binary is built and released by its own author. Sha256 verification protects against in-flight tampering of the declared archive; it does not vouch for the plugin author. Treat krew-installed plugins as "third-party tools running with your kubeconfig's RBAC", because that is what they are.
- **Plugin updates are not automatic.** `kubectl krew upgrade` is a manual step. CI images that bake in krew should re-run `kubectl krew upgrade --all` on rebuild rather than expect rolling updates.
- **Some kubectl verbs are reserved.** krew refuses to install a plugin whose name collides with a built-in `kubectl` subcommand (e.g. you cannot have a plugin literally called `kubectl-get`). Custom plugin indexes can publish a plugin under a prefixed name (e.g. `acme-deploy`) to avoid collisions.
- **Storage layout under `~/.krew/store/`.** Each installed plugin lives at `~/.krew/store/<plugin>/<version>/`, and `~/.krew/bin/kubectl-<plugin>` is a symlink into the current version. Manual deletion of a directory in `store/` can leave a dangling symlink — use `kubectl krew uninstall` rather than `rm -rf`.
- Apache-2.0 ([LICENSE](https://github.com/kubernetes-sigs/krew/blob/master/LICENSE)) — permissive; safe to bake into operator images, ops runner containers, and bastion hosts.

## Example invocations

```bash
# Bootstrap krew itself (one-time)
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

# Discover and install plugins
kubectl krew search rbac
kubectl krew info access-matrix
kubectl krew install neat tree ctx ns view-secret access-matrix rolesum

# Use the freshly installed plugins as native kubectl verbs
kubectl ctx                        # switch context
kubectl ns kube-system             # switch namespace
kubectl get deploy -o yaml | kubectl neat
kubectl tree deploy/api -n prod
kubectl access-matrix --namespace prod

# List, upgrade, uninstall
kubectl krew list
kubectl krew upgrade
kubectl krew uninstall view-secret

# Add a private / org-internal plugin index
kubectl krew index add acme https://git.internal/acme/krew-index.git
kubectl krew install acme/prod-shell
```
