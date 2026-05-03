# vault

- **Repo:** https://github.com/hashicorp/vault
- **Version:** 2.0.0 (latest stable, 2026-04-14)
- **License:** BUSL-1.1 ([LICENSE](https://github.com/hashicorp/vault/blob/main/LICENSE)) — source-available, not OSI-approved; converts to MPL-2.0 four years after each release
- **Language:** Go
- **Install:** `brew install hashicorp/tap/vault` · `apt install vault` (HashiCorp apt repo) · `pacman -S vault` · static binary tarball from [releases.hashicorp.com](https://releases.hashicorp.com/vault/) · official `hashicorp/vault` Docker image · `kubectl apply` via the [Vault Helm chart](https://github.com/hashicorp/vault-helm) for the server

## What it does

`vault` is a single Go binary that does double duty as the **secret-management server** and the **operator/client CLI** that talks to it. The server stores arbitrary secrets (database passwords, API tokens, TLS keys, SSH host keys, cloud-IAM credentials) encrypted at rest in a pluggable storage backend (Raft integrated storage is the default since 1.4; Consul, S3, file, and others remain supported), gates every read/write through a per-token policy engine, and emits a tamper-evident audit log of every request. The CLI surface is a verb-noun tree: `vault kv put secret/app/prod db_url=postgres://…` writes a versioned KV secret; `vault kv get -field=db_url secret/app/prod` reads one field for shell substitution; `vault read database/creds/app-readonly` asks Vault to *generate a fresh database credential on demand* and hand back a 1-hour lease (the database secrets engine speaks Postgres, MySQL, Mongo, MSSQL, Cassandra, and ~20 others); `vault write pki/issue/web common_name=svc.example.com` mints a short-lived TLS cert from Vault's internal CA; `vault token create -policy=app -ttl=1h` issues a scoped client token; `vault login -method=oidc` logs an operator in via SSO. Authentication is itself pluggable — `userpass`, `approle` (the canonical machine-auth method), `kubernetes` (the pod's service-account JWT becomes the Vault identity), `aws`/`gcp`/`azure` IAM, `oidc`, `ldap`, `cert`, and `jwt` are all built-in. The dynamic-secrets and lease story is what separates Vault from "encrypted KV": Vault doesn't *store* a Postgres password; it *generates* one when an app asks, hands it out with a TTL, and revokes it on lease expiry — so a leaked credential has bounded blast radius and the audit log shows exactly which token requested it. v1.4 added **Raft integrated storage** (no external Consul required for HA); v1.10 added **OIDC provider** mode (Vault as IdP); v1.13 added **plugin multiplexing**; v1.15 added **transit BYOK**; v2.0 (the current release) finalized the long-running plugin-runtime overhaul and ships first-class WIF (workload-identity federation) for the cloud auth methods. The CLI also speaks the HTTP API directly (`vault read sys/health`) and most production callers use the API over HTTPS rather than spawning the CLI.

## When to pick it / when not to

Pick `vault` when (a) you need **dynamic, short-lived credentials** for databases, cloud IAM, SSH, or PKI rather than long-lived static secrets checked into a config repo; (b) you need a **single audited choke point** for "who read which secret when" across a fleet of services; (c) you already run a HashiCorp stack (`terraform`, `consul`, `nomad`) and want the same auth/policy model end-to-end; (d) you have a compliance posture (SOC 2, PCI, HIPAA, FedRAMP) that asks for centralized secret rotation and audit trails. Common production shapes: a Kubernetes cluster where every pod authenticates to Vault via its service-account JWT and pulls fresh DB credentials at startup; a CI runner pool where `vault write aws/sts/deploy ttl=15m` mints a short-lived AWS session for each deploy; an internal PKI where `vault write pki/issue/internal` replaces a pet ACME setup with leaf certs that live for hours, not months; a developer-laptop workflow where `vault login -method=oidc` followed by `vault kv get` replaces ad-hoc 1Password sharing of staging credentials. Pair with [`consul`](../consul/) for service-discovery integration; pair with [`nomad`](../nomad/) for workload-identity-based secret injection; pair with [`terraform`](https://www.terraform.io/) (or [`tofu`](../tofu/) on the OSS side) for declarative policy/auth-method management; pair with [`step-cli`](../step-cli/) when the use case is *just* an internal PKI and Vault feels like overkill.

Skip Vault when the secrets footprint is genuinely small (a five-service startup with a handful of API keys is better served by SOPS-encrypted files in git, [`sops`](../sops/) + age, or a managed cloud KMS like AWS Secrets Manager / GCP Secret Manager). Skip when the team has no operational appetite for an HA stateful service (Vault unsealed-state, snapshot, and Raft quorum management are real ops surface — a misconfigured seal-stanza or a corrupted Raft snapshot will pin the cluster). Skip when license posture is a hard blocker: Vault moved from MPL-2.0 to **BUSL-1.1** in August 2023; the BUSL is *not* OSI-approved and forbids "competing offerings" — if your legal team requires an OSI-approved upstream, look at **[OpenBao](https://github.com/openbao/openbao)**, the Linux-Foundation MPL-2.0 fork of the last MPL Vault release. Skip when you only need encryption-at-rest for a static KV blob (cloud KMS + envelope encryption is simpler).

## Vs already cataloged

- **Vs [`sops`](../sops/) + [`age`](../age/) / [`rage`](../rage/):** different shape entirely. `sops` encrypts secrets *in git* and decrypts at deploy time using a KMS key — there is no running server, no audit log, no dynamic generation, no leases. Pick `sops` when secrets are few, slow-moving, and a checked-in encrypted file plus IAM on the KMS key is the threat model you want. Pick Vault when secrets are many, short-lived, machine-generated, and the audit log is a compliance requirement.
- **Vs [`teller`](../teller/):** teller is a *client-side aggregator* — it pulls secrets from many backends (Vault included, plus AWS/GCP/Azure secret managers, 1Password, Doppler) and injects them as env vars for a child process. Use teller as the developer-laptop / CI front-end on top of a Vault (or other) backend.
- **Vs [`ggshield`](../ggshield/) / [`gitleaks`](../gitleaks/) / [`trufflehog`](../trufflehog/) / [`noseyparker`](../noseyparker/) / [`ripsecrets`](../ripsecrets/):** orthogonal. Those scan code/history *for leaked* secrets. Vault is the system you adopt so secrets aren't sitting in code in the first place.
- **Vs [`infisical`](https://github.com/Infisical/infisical) (not in catalog) / Doppler / 1Password Connect:** modern KV-secrets-as-a-service alternatives with friendlier UX for developer-only use cases but smaller dynamic-secrets / PKI / SSH-CA story than Vault.
- **Vs [`step-cli`](../step-cli/):** step is a focused private-CA / PKI / SSH-CA toolkit. Vault's `pki` and `ssh` secrets engines overlap step substantially. Pick step when PKI is the *only* thing you need — the operational surface is much smaller. Pick Vault when PKI is one of many secret types you manage.
- **Vs [`cosign`](../cosign/) / [`gitsign`](../gitsign/) / [`witness`](../witness/) / [`rekor`](../rekor/):** different problem (artifact signing & supply chain) — Vault's `transit` engine can sign blobs but is not a Sigstore replacement.
- **Vs OpenBao:** OpenBao is the OSI-licensed (MPL-2.0) fork of Vault forked from the last pre-BUSL release. CLI flags and HTTP API are wire-compatible for the v1.x feature set. Pick OpenBao when license posture matters; pick upstream Vault when you need features added after the fork (Vault v2.0 has features OpenBao does not).

## Caveats

- **License is BUSL-1.1, not OSI-approved.** Production use inside a single company is allowed; offering Vault as a hosted service that competes with HashiCorp's own offering is not. Each release flips to MPL-2.0 four years after publication. If your org has a hard "OSI only" rule, use OpenBao instead.
- **Unseal is real operational surface.** A freshly started Vault server is *sealed* — its master key is encrypted and the server cannot decrypt secrets until it is unsealed. Production deployments use *auto-unseal* via cloud KMS (AWS KMS, GCP KMS, Azure Key Vault, or Transit-Auto-Unseal against another Vault). Shamir's-secret-sharing manual unseal is for dev clusters and disaster recovery only. Plan auto-unseal *before* you go to production.
- **Tokens are bearer credentials.** Anyone who reads a valid Vault token from `~/.vault-token`, an env var, or a process tree can use it until it expires. Use short TTLs (`-ttl=1h`), prefer AppRole or workload-identity auth for machines, and never log token values.
- **Audit log is mandatory for compliance use.** Vault writes audit log entries *before* completing the request, so if no audit device is configured (or all devices are failing), Vault refuses requests. Configure at least two audit devices (file + syslog, or two separate file paths) so a single FS issue doesn't take the cluster down.
- **Storage backend choice is permanent in practice.** Migrating from Consul-backed storage to Raft-integrated storage requires the documented `vault operator migrate` flow; doing it sloppily on a populated cluster is how teams lose secrets. Pick Raft for new deployments unless you have a specific reason not to.
- **Plugin compatibility is per-Vault-version.** Custom secret/auth plugins must be re-tested on each Vault upgrade — the plugin runtime contract is stable but not frozen, and v2.0 in particular tightened plugin multiplexing semantics.
- **The CLI reads `VAULT_ADDR` and `VAULT_TOKEN` from the environment.** A common foot-gun is running `vault` against the wrong cluster because a stale `VAULT_ADDR` is exported from a shell rc — print `vault status` first when you're not sure which cluster you're talking to.
- BUSL-1.1 ([LICENSE](https://github.com/hashicorp/vault/blob/main/LICENSE)). The Vault Go *client SDK* (`github.com/hashicorp/vault/api`) and several sub-packages remain MPL-2.0 — only the server and CLI are BUSL.

## Example invocations

```bash
# Install
brew install hashicorp/tap/vault                       # macOS
sudo apt install vault                                 # Debian/Ubuntu (HashiCorp apt repo)

# Point CLI at a server
export VAULT_ADDR='https://vault.example.com:8200'
vault status                                           # is this cluster up + unsealed?

# Log in (interactive)
vault login -method=oidc                               # SSO via configured OIDC provider
vault login -method=userpass username=alice            # username/password

# KV v2 (the most common path)
vault kv put   secret/app/prod db_url='postgres://…' api_key='sk-…'
vault kv get   secret/app/prod
vault kv get   -field=db_url secret/app/prod           # for shell substitution
vault kv get   -version=3   secret/app/prod            # versioned read
vault kv patch secret/app/prod feature_flag=true       # field-level update
vault kv delete secret/app/prod                        # soft-delete (recoverable)

# Dynamic database credentials (TTL-bounded)
vault read database/creds/app-readonly                 # mint a fresh Postgres user

# Short-lived AWS session for a CI deploy
vault write -force aws/sts/deploy ttl=15m

# Internal PKI: issue a leaf cert valid for 12h
vault write pki/issue/internal common_name=svc.example.com ttl=12h

# Token management
vault token create -policy=app -ttl=1h -renewable=true
vault token revoke <token-accessor>

# Policy authoring
vault policy write app - <<'EOF'
path "secret/data/app/*" { capabilities = ["read"] }
path "database/creds/app-readonly" { capabilities = ["read"] }
EOF

# Audit + health
vault audit enable file file_path=/var/log/vault_audit.log
vault operator raft list-peers                         # Raft cluster topology
vault operator step-down                               # graceful leader handoff
```
