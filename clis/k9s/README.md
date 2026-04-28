# k9s

- **Repo:** https://github.com/derailed/k9s
- **Version:** v0.50.18 (latest stable, 2026-04)
- **License:** Apache-2.0 ([LICENSE](https://github.com/derailed/k9s/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install derailed/k9s/k9s` · `go install github.com/derailed/k9s@latest` · binary releases on the GitHub release page

## What it does

`k9s` is a full-screen terminal UI for managing Kubernetes clusters. It reads
your active kubeconfig context (respecting `$KUBECONFIG`) and renders a
keyboard-driven, vim-flavoured TUI over the live cluster: `:pods`,
`:deployments`, `:svc`, `:nodes`, `:cm`, `:secrets`, `:events`, `:ns`, `:ctx`
(and the same for any CRD installed in the cluster) each open a sortable,
filterable, auto-refreshing view. From the row under the cursor you tail logs
(`l`), exec into a container shell (`s`), describe (`d`), edit live (`e`),
port-forward (`Shift+F`), delete with confirmation (`Ctrl+D`), or pop a YAML
view (`y`). Resource usage (CPU/mem) renders inline if `metrics-server` is
installed. Pulse mode (`:pulse`) shows real-time cluster activity, XRay
(`:xray`) shows owner-reference graphs, and the popeye plugin highlights
misconfigurations. Customisation is YAML in `~/.config/k9s/`: skins, hotkeys,
plugins, aliases, and per-context views.

## When to pick it / when not to

Pick `k9s` when you spend any meaningful time inspecting a live cluster from
the terminal and `kubectl get pods -w | grep ... | awk ...` plus a second
window for `kubectl logs -f` plus a third for `kubectl describe` is the loop
you would otherwise run. The single-screen, auto-refresh, one-keystroke-action
model collapses 4-6 `kubectl` invocations into one TUI session and is the
canonical answer for "I need to triage what is wrong in this cluster right
now". Pairs naturally with [`kdash`](../kdash/) (lighter Rust alternative,
overview-first) and orthogonal to [`kubectl-ai`](../kubectl-ai/) which is the
LLM-driven natural-language layer over the same `kubectl` API.

Skip it when the workflow is one-shot scripted (`kubectl apply -f` in CI —
stay on raw `kubectl`), when you want a read-only overview without per-row
mutating actions (use `kdash` or the dashboard add-on), or when the cluster
is a single-node minikube where the TUI overhead is more than the workload it
manages. Default keybinds give powerful destructive actions (`Ctrl+D` delete,
`e` live edit) two keystrokes deep — for production clusters, set up
read-only contexts via RBAC rather than relying on muscle memory, and consider
the `--readonly` flag.

## Example invocations

```bash
# Open against the active kubeconfig context, default namespace
k9s

# Pin to a specific namespace and start in the pods view
k9s -n kube-system -c pod

# Read-only mode for a production cluster (disables all mutating actions)
k9s --readonly --context prod-eu-west-1

# Use a specific kubeconfig file
k9s --kubeconfig ~/.kube/staging.yaml

# Inside k9s: filter pods by label, then tail logs
# :pods<Enter>  /app=api<Enter>   l   (then Esc to detach)
```
