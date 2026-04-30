# kubeseal

> **Client CLI for Sealed Secrets — encrypt Kubernetes Secrets
> so the ciphertext is safe to commit to git.** Pairs with the
> in-cluster `sealed-secrets` controller (same repo) which
> holds an asymmetric keypair: `kubeseal` fetches the public
> key, encrypts a regular `Secret` manifest into a
> `SealedSecret` custom resource, and only the controller
> running in the target cluster / namespace can decrypt it
> back into a `Secret`. Pinned to **v0.36.6**
> ([LICENSE](https://github.com/bitnami-labs/sealed-secrets/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/bitnami-labs/sealed-secrets>

## TL;DR

Write a normal `Secret` YAML, pipe it through `kubeseal -o
yaml > sealed.yaml`, and commit `sealed.yaml`. The resulting
`SealedSecret` is a regular Kubernetes object with an opaque
`encryptedData` blob — applying it via `kubectl apply` /
ArgoCD / Flux causes the in-cluster controller to decrypt and
materialise the underlying `Secret`. Default scope binds the
ciphertext to the `{namespace, name}` pair so it cannot be
re-used elsewhere; `--scope namespace-wide` and `--scope
cluster-wide` relax that. Key rotation is automatic
(controller adds a fresh key every 30 days, keeps old keys
for decryption), and `kubeseal --re-encrypt` rolls existing
SealedSecrets onto the latest key.

## Install

```bash
# Homebrew (macOS / Linux) — installs the kubeseal CLI
brew install kubeseal

# Direct binary
curl -sSL https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.36.6/kubeseal-0.36.6-linux-amd64.tar.gz \
  | tar xz kubeseal && sudo mv kubeseal /usr/local/bin/

# Install the in-cluster controller (Helm)
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  -n kube-system
```

## Example

```bash
# Encrypt a Secret manifest and commit the result
kubectl create secret generic db-creds \
  --from-literal=password='hunter2' \
  --dry-run=client -o yaml \
  | kubeseal -o yaml > k8s/db-creds.sealed.yaml

# Re-key all existing SealedSecrets onto the controller's
# current key (e.g. after rotation)
kubectl get sealedsecrets -A -o yaml \
  | kubeseal --re-encrypt -o yaml \
  > all-sealed-resealed.yaml

# Fetch and pin the controller's public cert for offline use
kubeseal --fetch-cert > pub-cert.pem
kubeseal --cert pub-cert.pem -o yaml < secret.yaml
```

## When to use

- You want secrets in git for GitOps reconciliation but cannot
  commit cleartext.
- You run multiple clusters and want each cluster's secrets
  bound to *that* cluster's key (so a leaked repo cannot leak
  prod secrets into staging).
- You want a self-hosted, dependency-free alternative to
  external KMS / Vault / cloud secret managers for the
  Kubernetes-only case.

## When NOT to use

- You already run Vault / cloud KMS and want secrets injected
  at runtime (sidecar / CSI driver) rather than materialised
  as Kubernetes Secrets.
- You need *application-level* encryption that survives even
  a cluster-admin compromise; SealedSecrets become plain
  `Secret`s once decrypted in the cluster.
- You need to share one ciphertext across many namespaces
  without `--scope cluster-wide` (which weakens the binding
  guarantee).
