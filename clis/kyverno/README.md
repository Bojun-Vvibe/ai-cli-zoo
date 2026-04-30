# kyverno

> **Kubernetes-native policy engine without a separate DSL.** Policies
> are themselves Kubernetes resources (`ClusterPolicy` /
> `Policy` / `ValidatingPolicy` / `ImageVerificationPolicy`) written
> in plain YAML pattern-matching, so admission validation, mutation,
> generation, image-signature verification, and cleanup live as CRDs
> the cluster reconciles like any other workload. The `kyverno` CLI
> tests, applies, and lints those policies offline (CI gate) and
> against live clusters. Pinned to **v1.18.0**
> ([LICENSE](https://github.com/kyverno/kyverno/blob/v1.18.0/LICENSE),
> Apache-2.0).

Source: <https://github.com/kyverno/kyverno>

## TL;DR

`kyverno` is the CNCF-graduated policy engine whose pitch is "no
Rego" — instead of writing OPA/Gatekeeper Rego, you write a
`ClusterPolicy` whose `match` block selects resources and whose
rules use a YAML overlay-pattern syntax (`pattern:` for validation,
`mutate:` with strategic-merge or JSON-patch for defaulting,
`generate:` for cascading object creation, `verifyImages:` for
signature enforcement against Sigstore / Notary). The cluster
controller registers as an admission webhook, so policies block /
mutate at the API server, and the CLI runs the same engine offline
against a directory of manifests + a directory of policies for CI
gating before the manifests ever reach a cluster. Built-in policy
exceptions (`PolicyException` CRD) let platform teams grant scoped
waivers without editing the policy itself, and the `PolicyReport`
CRD surfaces audit-mode findings as queryable cluster state for
dashboards. The companion `kyverno-json` engine extends the same
pattern syntax to arbitrary JSON / Terraform / CloudFormation
payloads, so the same engine gates IaC at PR time and runtime
manifests at admission time.

## Install

```bash
# Homebrew (macOS / Linux)
brew install kyverno

# Krew kubectl plugin
kubectl krew install kyverno

# Install the cluster controllers via Helm
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno --version 3.5.4 \
  --namespace kyverno --create-namespace

# Apply the curated baseline / restricted PSS policies
kubectl apply -f https://github.com/kyverno/policies/releases/download/stable/install.yaml
```

## Example

```bash
# Lint a directory of policies against the engine schema
kyverno fix policy ./policies/

# CI gate: run policies against a manifests directory offline
kyverno apply ./policies/ --resource ./manifests/ \
  --policy-report --output-format=junit > kyverno-report.xml

# Test a policy with declared expected results (kyverno-test.yaml)
kyverno test ./policies/disallow-latest-tag/

# Verify image signatures locally before commit
kyverno apply ./policies/verify-images.yaml \
  --resource ./manifests/deployment.yaml --cluster=false

# Inspect policy reports already produced by the in-cluster controller
kubectl get policyreports -A
kubectl get clusterpolicyreports
```

## When to use

- You want admission policy as plain Kubernetes YAML reviewed by
  the same teams that review manifests, with no Rego onboarding.
- You need mutation + generation in the same engine as validation
  (default `runAsNonRoot: true`, generate a `NetworkPolicy` for
  every new namespace) — Gatekeeper validates only.
- You want one policy set that gates IaC at PR time
  (`kyverno apply` in CI) and live manifests at admission time
  (the controller webhook).

## When NOT to use

- Your platform already standardised on OPA / Gatekeeper and your
  Rego library is large — switching costs outweigh DSL ergonomics.
- You need policy decisions outside Kubernetes (data plane, app
  authz) — that's [`opa`](https://www.openpolicyagent.org) /
  [`cedar`](https://www.cedarpolicy.com) territory; Kyverno's home
  is the Kubernetes admission path.
- You only need PodSecurity baselines — the in-tree `PodSecurity`
  admission plugin covers PSS without an extra controller.
