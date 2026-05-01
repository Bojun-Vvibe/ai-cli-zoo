# teller

- **Repo:** https://github.com/tellerops/teller
- **Version:** v2.0.7 (Rust rewrite line)
- **License:** Apache-2.0 ([LICENSE.txt](https://github.com/tellerops/teller/blob/master/LICENSE.txt))
- **Language:** Rust (v2.x); originally Go (v1.x)
- **Install:** `brew install tellerops/tap/teller` · `cargo install teller-cli` · prebuilt binaries on the GitHub releases page · `curl -fsSL https://raw.githubusercontent.com/tellerops/teller/master/install.sh | sh`

## What it does

`teller` is a single static binary that reads a small
declarative file (`.teller.yml`) describing where each secret
your app needs lives — across Vault, AWS Secrets Manager, AWS
SSM Parameter Store, GCP Secret Manager, Azure Key Vault,
1Password Connect, HashiCorp Consul, Doppler, etcd, dotenv
files, LastPass, CyberArk Conjur, Heroku config, Cloudflare
Workers env, Google Cloud KMS, and ~20 other providers — and
then materialises those secrets on demand into a child process
without ever writing them to disk. The two load-bearing verbs
are `teller run -- <cmd> <args>`, which fetches every mapping
from the active providers, exports them as env vars in a
spawned child, and exits when the child exits (so a misbehaving
agent / dev shell / CI step inherits credentials only for its
lifetime), and `teller export <fmt>` (`env` / `dotenv` /
`yaml` / `json` / `csv` / `k8s` / `terraform` / `cloudflare`)
which prints the same materialised set in whichever shape the
downstream tool expects. Companion verbs cover the rest of the
secret lifecycle: `teller scan` greps a tree for literal
secret values that should be coming from a provider (a
pre-commit-friendly gate), `teller redact` filters stdin /
stdout to mask known secret values from logs in real time,
`teller copy --from src --to dst` moves secrets between
providers (Vault → AWS SSM during a migration), `teller mirror
--from src --to dst` keeps a one-way sync going, and `teller
delete` / `teller put` cover scripted CRUD against any
configured provider through one uniform interface. Output of
`teller env` is shell-eval-safe (`eval "$(teller env)"`) and
`teller graph` renders a DOT graph of which providers each
mapping touches, useful for compliance review.

## What's interesting

- **One CLI, ~20 secret backends** — the same `.teller.yml`
  schema covers Vault, AWS SM/SSM, GCP SM, Azure KV,
  1Password, Doppler, Consul, etcd, dotenv, etc.; switching
  backends per environment is a YAML edit, not a code change.
- **`teller run -- <cmd>` injects then exits** — the child
  inherits credentials in env, the parent process sees nothing
  on disk; mirrors `direnv` semantics but pulls from real
  secret managers.
- **`teller scan` + `teller redact`** — pre-commit gate that
  catches leaked plaintext copies of provider-managed secrets,
  plus a streaming log filter that masks known values out of
  CI output and shell history.
- **`teller copy` / `teller mirror`** — provider-to-provider
  migration / replication is one command, so "move from
  HashiCorp Vault to AWS Secrets Manager" doesn't need a
  custom script.
- **Format-agnostic export** — same fetched set renders as
  POSIX env, dotenv, YAML, JSON, k8s `Secret` manifest, or
  Terraform variables, so downstream tools each get the shape
  they want from one canonical source.

## AI-native angle

LLM-driven dev loops have a chronic credential-handling
problem: an agent that needs `OPENAI_API_KEY`, `GITHUB_TOKEN`,
a database URL, and a Vault token to do its job ends up with
all four either pasted into a prompt or written to a `.env`
file the agent itself can read and exfiltrate via the next
tool call. `teller run -- <agent-cmd>` cuts that surface area
to zero by fetching from the org's actual secret manager and
handing the agent only the env vars its forked process needs,
for only as long as the process runs — no disk artefact, no
prompt content, no agent-readable state file. Pair with a
narrow allowlist in `.teller.yml` per agent role
(planner-agent gets read-only API keys, executor-agent gets
write tokens) and the same governance model your humans use
for production access automatically applies to the LLM. The
`teller redact` filter in the agent's stdout/stderr pipeline
prevents the model from echoing a secret it received back into
its own context window. Pairs with [`age`](../age/) /
[`sops`](../sops/) (file-level encryption for things that
*do* need to be on disk like committed encrypted manifests)
and [`gitleaks`](../gitleaks/) / [`trufflehog`](../trufflehog/)
(history-scanning for what already leaked).