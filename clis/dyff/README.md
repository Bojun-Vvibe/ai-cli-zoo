# dyff

> **A diff tool for YAML / JSON that understands structure —
> reorder a list, rename a key, change one nested value, and
> `dyff` shows three semantic changes instead of a 200-line
> textual diff.** Pinned to **v1.12.0**,
> [LICENSE](https://github.com/homeport/dyff/blob/main/LICENSE),
> MIT.

Source: <https://github.com/homeport/dyff>

## TL;DR

`dyff` is what you reach for when `diff -u a.yaml b.yaml` shows
80 lines of indentation noise for what is actually one renamed
key and a list reordering. It parses both sides as YAML / JSON,
walks the resulting trees, and emits a path-addressed list of
*structural* changes — `spec.template.spec.containers.[name=app].image
changed from foo:1.2.3 to foo:1.2.4` — with optional Spruce
dot-style or GoPatch path syntax. The killer use case is
diffing two rendered Kubernetes manifests (Helm `template`
output before and after a values change, or the live cluster
vs. a proposed `kubectl apply --dry-run=server -o yaml`
result) where line diffs are unreadable but structural diffs
are exactly the review surface a human wants.

## Install

```bash
# Homebrew (macOS / Linux)
brew install homeport/tap/dyff

# Pre-built binary (any platform)
curl -L "https://github.com/homeport/dyff/releases/download/v1.12.0/dyff_1.12.0_$(uname -s | tr A-Z a-z)_amd64.tar.gz" \
  | tar xz dyff && sudo mv dyff /usr/local/bin/

# Go install
go install github.com/homeport/dyff/cmd/dyff@v1.12.0

# verify
dyff version              # 1.12.0
dyff between --help
```

## License

MIT — see
[LICENSE](https://github.com/homeport/dyff/blob/main/LICENSE).
Permissive: ship the binary in CI images freely; no obligations
on the YAML you diff with it.

## One Concrete Example

A typical "what does this Helm values change actually do?"
review workflow:

```bash
# Render the chart with the old values and the new values.
helm template my-app ./charts/my-app -f values/prod.yaml         > before.yaml
helm template my-app ./charts/my-app -f values/prod.new.yaml     > after.yaml

# Structural diff. Output is colourised by default; --no-color in CI.
dyff between before.yaml after.yaml
```

Sample output (one rename, one image bump, one list reorder
that a textual diff would show as a 40-line rewrite):

```
     _        __  __
   _| |_   _ / _|/ _|  between before.yaml
 / _' | | | | |_| |_       and  after.yaml
| (_| | |_| |  _|  _|
 \__,_|\__, |_| |_|   returned three differences
        |___/

spec.template.spec.containers.app.image
  ± value change
    - quay.io/example/app:1.2.3
    + quay.io/example/app:1.2.4

spec.template.spec.containers.app.env.LOG_LEVEL
  ± value change
    - info
    + debug

spec.template.spec.containers.app.env (order changed)
  ⨁ list order changed
    - 1: AWS_REGION
    + 1: LOG_LEVEL
```

CI usage:

```bash
# Exit non-zero if there is any structural change (gating)
dyff between live.yaml proposed.yaml --set-exit-code

# JSON output for tools / bots
dyff between live.yaml proposed.yaml --output-style json | jq

# Diff a Kubernetes apply preview against the live state
kubectl get deployment app -o yaml > live.yaml
kubectl apply -f new.yaml --dry-run=server -o yaml > proposed.yaml
dyff between live.yaml proposed.yaml --ignore-order-changes
```

## Niche It Fills

**Structural diff for hierarchical config.** YAML is the
universal config format for Kubernetes / Helm / Compose /
GitHub Actions / Ansible / CI pipelines, and review of YAML
changes happens in PRs hundreds of times per week per team.
Plain `diff` / `git diff` operates on lines and is defeated
by anything that touches indentation: a `yq -i 'sort_keys(..)'`
run shows up as a full-file rewrite, a list reorder shows up
as the entire list deleted-and-readded, and a values
re-template can balloon a one-key change into 50 lines of
noise. `dyff` parses the structure and reports *what
actually changed*: which path, what value, what kind of
change (added / removed / modified / reordered). Pair it
with `helm template` / `kubectl apply --dry-run` / `kustomize
build` and "what does this PR do to the cluster?" becomes a
real question with a readable answer.

## Why use it

Three things `dyff` does that `diff -u` / `git diff` /
`yq` cannot, all at once:

1. **Path-addressed change reports.** Every change is
   labelled with its full structural path
   (`spec.template.spec.containers.[name=app].image`), not
   a line number. The path is stable across formatting
   changes — re-indenting the file does not invent new
   "differences," and a key moved 200 lines down is still
   recognised as the same key. Reviewers cite paths in PR
   comments instead of line ranges that drift on every
   rebase.
2. **List-order awareness, optional.** YAML lists where
   order does not matter (env vars, init-containers,
   labels, args) drive textual diff to its knees. `dyff
   --ignore-order-changes` treats reorders as no-ops; the
   default mode reports them explicitly as a separate
   change kind so you can see "this is just a sort, no
   semantics changed."
3. **Targeted comparisons via path filtering.** `dyff
   between a.yaml b.yaml --filter spec.template` only
   reports changes under one subtree — the right tool when
   diffing a giant rendered manifest and you only care
   about the deployment spec, not the autogenerated
   `metadata.annotations.deployment.kubernetes.io/revision`
   churn. Combine with `--exclude` to silence well-known
   noise paths.

For a platform / Kubernetes / GitOps workflow that already
uses `helm template`, `kustomize build`, `kubectl
diff`, or `kubectl apply --dry-run=server`, `dyff` is the
viewer that makes the output of those commands actually
reviewable.

## Vs Already Cataloged

- **Vs [`yq`](../yq/):** complementary; orthogonal jobs.
  `yq` *transforms* YAML (`yq '.spec.replicas = 4' file
  -i`); `dyff` *compares* two YAML trees and explains the
  delta. A common pairing is `yq -i` to make a change,
  then `dyff between original.yaml modified.yaml` to
  verify only the intended path moved.
- **Vs [`delta`](../delta/) (this round):** different
  layer. `delta` is a syntax-highlighting `git diff`
  pager — it makes textual line diffs prettier. `dyff`
  changes the *unit of comparison* from lines to nodes;
  it is what you reach for when even a beautifully
  highlighted line diff is the wrong shape. Use both:
  `delta` for code, `dyff` for structured config.
- **Vs `kubectl diff`:** complementary. `kubectl diff`
  shows the server-side dry-run delta in textual form;
  pipe both halves through `dyff` for the structural
  view (`kubectl diff -f manifest.yaml | tee
  /tmp/k.diff` then re-render both and `dyff between`).
- **Vs `helmfile diff` (this round):** different
  altitude. `helmfile diff` orchestrates `helm-diff`
  across many releases at once and prints unified
  patches per release. `dyff` is the underlying viewer
  shape you want for any *one* release's manifest pair;
  some teams pipe `helm-diff` output through `dyff` for
  the readable view.

## Caveats

- **YAML-document-aware, but stream order matters for
  multi-document files.** A `kind: List` of N documents
  diffs cleanly; a flat stream of `---`-separated
  documents is matched positionally by default. Use
  `--detect-kubernetes-entity-file` (or
  `--use-go-patch-style` paths) so resources are
  matched by `kind/namespace/name`, not by stream
  position — otherwise inserting a new resource at
  position 3 reports every later resource as "moved."
- **Anchors and aliases are resolved before diffing.**
  Two files that achieve the same value via different
  anchor structure are reported identical (usually what
  you want); two files where one uses `<<: *common`
  and the other inlines the keys also report identical.
  If you specifically need to diff the *raw* anchor
  graph, use a different tool.
- **JSON works but the killer features are
  YAML-shaped.** `dyff` will diff JSON happily, but
  `jq -S` + `git diff` is often enough for JSON because
  JSON does not have the indentation / list-order
  pathologies YAML has. Reach for `dyff` first when the
  inputs are YAML.
- **Output is human-shaped by default.** Machine
  consumers should pass `--output-style json` or
  `--output-style brief` and parse those — the default
  ASCII-art header and ANSI colourisation are review
  ergonomics, not a stable contract.
- **No file-tree diff mode.** `dyff between` takes two
  files (or `-` for stdin), not two directories. For
  diffing two rendered manifest trees, render each side
  to a single multi-document YAML first
  (`kustomize build dir-a > a.yaml; kustomize build
  dir-b > b.yaml; dyff between a.yaml b.yaml`).
