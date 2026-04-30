# flipt

- **Repo:** https://github.com/flipt-io/flipt
- **Version:** v2.9.0
- **License:** [LICENSE](https://github.com/flipt-io/flipt/blob/main/LICENSE) (GPL-3.0)
- **Category:** Self-hosted feature flag + experimentation server with a CLI

## What it is

`flipt` is a self-hosted feature flag and dynamic configuration service that
ships as a single Go binary plus a CLI. It exposes flags, segments, rules,
and rollouts over HTTP/gRPC, evaluates them server-side or via embedded
evaluation libraries, and stores state in SQLite/MySQL/Postgres or — in
GitOps mode — in a Git repo as plain YAML files. The CLI bootstraps a
local instance, validates flag definitions, imports/exports state, runs
the embedded evaluator against a flag set without a server, and operates
the bundled OFREP-compatible API for OpenFeature clients.

> Note on version: this entry pins **v2.9.0**, the latest tag at time of
> writing. Flipt v2 introduced a redesigned, GitOps-first storage model
> (flags live as YAML in a Git repo by default); the v1 line (latest
> `v1.61.1`) is still maintained for users who want the original
> database-backed deployment shape.

## Install

```
brew install flipt-io/brew/flipt                                # macOS / Linuxbrew
# or grab the binary from https://github.com/flipt-io/flipt/releases
flipt --version
```

## Basic usage

```
flipt                                          # start server (UI on :8080, gRPC on :9000)
flipt config init                              # write a default config.yaml
flipt validate features.yaml                   # lint a flag definition file
flipt import features.yaml                     # load flags into the configured backend
flipt export -o features.yaml                  # snapshot current state to YAML
flipt evaluate --flag my-flag --entity user-42 # one-shot evaluation from the CLI
```

## When to use it

- You want **feature flags without sending every evaluation to a SaaS
  vendor** — for compliance, latency, air-gapped envs, or just cost.
- You need an **OpenFeature-compatible backend** so application code stays
  vendor-neutral and you can swap providers later.
- You like **flags-as-code in a Git repo** (v2 GitOps mode) so flag changes
  go through the same review/PR flow as application code.
- Skip it when you genuinely need **multi-region SaaS scale, A/B stats
  pipelines, and audience targeting out of the box** — a hosted product
  (LaunchDarkly, Statsig, Unleash Cloud) will get you there faster than
  operating Flipt yourself.
- Note: the server is **GPL-3.0**, which matters if you plan to distribute
  a modified Flipt binary to third parties; ordinary in-house operation and
  evaluation via the network APIs are unaffected.
