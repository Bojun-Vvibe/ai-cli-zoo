# esc

> **Pulumi ESC — Environments, Secrets, and Configuration as a
> centralized service** with a stand-alone `esc` CLI that resolves
> hierarchical, composable environment definitions (YAML with
> imports, interpolation, and dynamic-credential providers) and
> exposes them as plain env vars, files, or a secrets-broker URL
> that any CLI can consume — independent of the broader Pulumi
> IaC product.
> Pinned to **v0.23.0** (released 2026-03-12,
> [`gh api repos/pulumi/esc/releases/latest`](https://github.com/pulumi/esc/releases/latest),
> [LICENSE](https://github.com/pulumi/esc/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/pulumi/esc>

## TL;DR

Most secret-management tools answer one of two questions: **"how
do I store a secret encrypted-at-rest?"** ([`sops`](../sops/),
[`vault`](../vault/), [`kubeseal`](../kubeseal/),
[`gopass`](../gopass/)) or **"how do I get short-lived cloud
creds?"** ([`aws-vault`](../aws-vault/),
[`granted`](../granted/)). Pulumi ESC answers a third, increasingly
common question: **"how do I compose one logical *environment*
(staging, prod-eu, ci) out of secrets in three different stores
plus dynamic AWS / Azure / GCP credentials plus a few literal
config values, version it, and hand it to *any* CLI / pipeline /
shell unchanged?"** An environment in ESC is a YAML doc with three
superpowers: `imports:` lets one env extend others (so `prod-eu`
imports `prod-base` imports `org-base`), `${...}` interpolation
references other keys or imported envs, and `fn::open::*`
*providers* synthesize values at read-time (open an AWS OIDC
session, fetch a Vault path, read a 1Password item, decrypt a
KMS-encrypted blob). The `esc` CLI then projects the resolved
environment as `--env-vars` for `esc run`, a dotenv file for
`esc env open`, or a typed JSON document for programmatic use.
You can run ESC against the hosted Pulumi Cloud backend (free tier
covers individuals + small teams) or self-host the open-source
service. The CLI is Apache-2.0 and works fully without the broader
Pulumi IaC tool.

## Install

```bash
# Standalone install script (the recommended path)
curl -fsSL https://get.pulumi.com/esc/install.sh | sh

# Homebrew
brew install pulumi/tap/esc

# Pre-built tarball
curl -LO https://github.com/pulumi/esc/releases/download/v0.23.0/esc-v0.23.0-linux-x64.tar.gz
tar xzf esc-v0.23.0-linux-x64.tar.gz && sudo mv esc/esc /usr/local/bin/

# Log in (Pulumi Cloud free account is enough for personal use;
# self-hosted backend is also supported)
esc login

# verify
esc version
```

## Representative examples

```bash
# 1. Define an environment (YAML) — composes a base env, pulls a
#    short-lived AWS role via OIDC, and decrypts a KMS secret
#    envs/staging.yaml
#    imports:
#      - org-base
#    values:
#      aws:
#        login:
#          fn::open::aws-login:
#            oidc:
#              roleArn: arn:aws:iam::123:role/staging
#              sessionName: esc
#      database:
#        url:
#          fn::secret:
#            ciphertext: ZXNjAAA...
#      environmentVariables:
#        AWS_ACCESS_KEY_ID:     ${aws.login.accessKeyId}
#        AWS_SECRET_ACCESS_KEY: ${aws.login.secretAccessKey}
#        AWS_SESSION_TOKEN:     ${aws.login.sessionToken}
#        DATABASE_URL:          ${database.url}
esc env init my-org/staging
esc env edit  my-org/staging   # opens $EDITOR

# 2. Print a resolved environment (secrets shown only with --show-secrets)
esc env open my-org/staging
esc env open my-org/staging --format=dotenv > .env.staging
esc env open my-org/staging --format=json   | jq

# 3. Run an arbitrary CLI with the resolved env as process env vars
#    (this is the everyday use — `esc run` is the moral equivalent of
#    `aws-vault exec`, but it works for *every* cloud + every secret
#    backend you wired up, not just AWS profiles)
esc run my-org/staging -- terraform plan
esc run my-org/staging -- ./deploy.sh
esc run my-org/staging -- env | grep DATABASE_URL

# 4. Compose envs (DRY across tiers)
#    envs/org-base.yaml
#    values:
#      observability:
#        OTEL_EXPORTER_OTLP_ENDPOINT: https://otel.example.com
#    envs/prod-eu.yaml
#    imports: [org-base, prod-base]
#    values:
#      database: { url: ${prod-base.database.url} }

# 5. Diff two environments (what changed between staging and prod?)
esc env diff my-org/staging my-org/prod

# 6. Version + rollback (every save is a numbered revision)
esc env version history my-org/prod
esc env version rollback my-org/prod 42
esc env tag      my-org/prod stable @42
```

## When to use vs. alternatives

- Pick **esc** when the *composition* problem is the actual pain:
  one logical environment is the union of values from KMS, Vault,
  cloud OIDC, and literals; multiple tiers (org / team / app /
  stage) need to inherit and override; and many tools (Terraform,
  shell scripts, Docker, K8s manifests, CI) need to consume the
  *same* resolved view. Especially good when secrets live in
  multiple backends and you don't want to migrate everything to
  one store.
- Pick [`sops`](../sops/) when the workflow is **"encrypted YAML/
  JSON files committed to Git, decrypted by KMS at deploy time"**
  and you don't need composition or dynamic credentials. `sops`
  can be wired *into* an ESC env as a provider — the two compose.
- Pick [`teller`](../teller/) for an OSS, no-backend-server,
  fan-out reader across many secret stores into env vars. ESC
  layers composition + revisions + a hosted service on top; teller
  is the simpler "just expose the secrets" path.
- Pick [`vault`](../vault/) when you need a full secrets *engine*
  with dynamic database creds, PKI, transit encryption, and policy
  — Vault is the storage + identity + leasing layer; ESC is a
  composition / projection layer that can read *from* Vault.
- Pick [`aws-vault`](../aws-vault/) /
  [`granted`](../granted/) for AWS-only short-lived creds — same
  job ESC does for AWS via `fn::open::aws-login`, but
  AWS-specific and lighter weight if that's the entire scope.
- Pick [`kubeseal`](../kubeseal/) when the consumer is *only*
  Kubernetes and SealedSecret CRDs in Git is enough. ESC is for
  the broader case where consumers include Terraform, Docker,
  shell, and CI — not just `kubectl apply`.
- Pick [`gopass`](../gopass/) for *team password management*
  (passwords, API keys for human use, GPG-encrypted on disk). ESC
  targets *machine* consumption.
- Pick [`dotenvx`](../dotenvx/) for the simplest case: encrypted
  `.env` files in Git, decrypted at process start. No composition,
  no providers, no service — but zero infrastructure too.
- Caveats: hosted backend ties you to Pulumi Cloud (free tier
  covers individuals; self-hosted is supported but you operate
  it), the YAML+CUE-flavoured composition is its own learning
  curve, and the project is pre-1.0 — APIs and the env spec are
  stable but pin a `v0.x` minor and read release notes before
  bumping. Despite the name, it does *not* require Pulumi IaC —
  the [`pulumi`](../pulumi/) entry covers that separately.
