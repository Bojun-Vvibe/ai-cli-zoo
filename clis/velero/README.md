# velero

> **Backup, restore, and migration for Kubernetes cluster
> resources and persistent volumes.** Ships a CLI plus an
> in-cluster controller that snapshots Kubernetes API objects
> (any group, any namespace) to object storage (S3, GCS, Azure
> Blob, MinIO, etc.) and orchestrates volume snapshots through
> CSI or per-provider plugins (AWS EBS, GCP PD, Azure Disk,
> Restic / Kopia for filesystem-level backups). Supports
> scheduled backups, selective restore, and cross-cluster
> migration. Pinned to **v1.18.0**
> ([LICENSE](https://github.com/vmware-tanzu/velero/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/vmware-tanzu/velero>

## TL;DR

`velero install` deploys the controller; `velero backup create
nightly --include-namespaces prod` snapshots every API object
in `prod` plus its PVs to the configured bucket. `velero
restore create --from-backup nightly` brings them back — into
the same cluster (disaster recovery) or a different one
(migration / blue-green). `velero schedule create daily
--schedule "0 2 * * *"` turns it into a cron. The Restic /
Kopia integration covers volumes whose CSI driver does not
support snapshots, so it works on bare-metal and edge
clusters too.

## Install

```bash
# Homebrew (macOS / Linux)
brew install velero

# Direct binary
curl -sSL https://github.com/vmware-tanzu/velero/releases/download/v1.18.0/velero-v1.18.0-linux-amd64.tar.gz \
  | tar xz && sudo mv velero-v1.18.0-linux-amd64/velero /usr/local/bin/

# In-cluster install (S3-compatible bucket, AWS plugin)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.0 \
  --bucket my-velero-bucket \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero
```

## Example

```bash
# One-shot backup of two namespaces
velero backup create app-pre-upgrade \
  --include-namespaces app,app-data \
  --wait

# Inspect what was captured
velero backup describe app-pre-upgrade --details

# Restore into a fresh namespace mapping
velero restore create app-restore \
  --from-backup app-pre-upgrade \
  --namespace-mappings app:app-recovered

# Cross-cluster migration: point a second cluster at the same
# bucket and `velero restore create --from-backup ...`
```

## When to use

- You need cluster-level disaster recovery for Kubernetes API
  objects *and* the PVs behind them, not just GitOps
  re-application of manifests.
- You are migrating workloads between clusters / cloud
  providers and want one tool to move objects + data.
- You want scheduled, retention-managed backups stored in your
  own object bucket rather than a SaaS backup service.

## When NOT to use

- You only need application-level backups (Postgres dumps,
  Mongo snapshots) — application-native tooling is usually
  faster and more consistent.
- Your cluster is fully GitOps-reconciled and stateless;
  re-applying manifests + restoring databases out-of-band may
  be simpler than introducing Velero.
- You need point-in-time consistency across many PVs at once;
  Velero coordinates per-volume snapshots, not a global
  freeze.
