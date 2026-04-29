# dotenvx

- **Repo:** https://github.com/dotenvx/dotenvx
- **Homepage:** https://dotenvx.com/docs
- **Version:** v1.64.0 (2026-04-27)
- **License:** BSD-3-Clause — [LICENSE](https://github.com/dotenvx/dotenvx/blob/main/LICENSE) (SPDX: `BSD-3-Clause`)
- **Language:** JavaScript (Node.js) — distributed as a single static binary built with `pkg`/Bun for every major OS / arch (no Node runtime required)
- **Install:** `brew install dotenvx/brew/dotenvx` · `curl -fsS https://dotenvx.sh \| sh` · `npm install -g @dotenvx/dotenvx` · `winget install dotenvx.dotenvx` · prebuilt single-binary tarballs on the GitHub release page · binary name is `dotenvx`

## What it does

`dotenvx` is the secure, modern successor to the original `dotenv` — by
the same author (Mot, of `motdotenv` / `dotenv`) — that makes a
`.env` file safe to commit to the repo by *encrypting* the values with
a public key that lives in `.env.vault` / `.env.keys`, decrypting them
just-in-time when a process starts.

- **Run anything with envs injected** — `dotenvx run -- node index.js`
  loads `.env` (and `.env.<NODE_ENV>` overlays, and per-environment
  encrypted overlays), expands `${VAR}` references, and execs the
  command with the resulting environment, so the application code stays
  free of any `require('dotenv').config()` line.
- **Encrypted .env files**, the headline feature — `dotenvx encrypt`
  rewrites every value in `.env` as `encrypted:BASE64...` using a
  per-environment public key and writes the matching private key to
  `.env.keys` (which stays out of git), so the encrypted `.env` is safe
  to commit and ship through CI without a separate secret-manager round
  trip; the runtime decryption only needs `DOTENV_PRIVATE_KEY` (or a
  `DOTENV_PRIVATE_KEY_PRODUCTION` for the prod overlay) in the deployment
  environment.
- **Drop-in for the original `dotenv`** — `dotenvx run -- npm start`
  replaces `node -r dotenv/config index.js`, no application code change,
  but you also get encrypted values, multi-env overlays, `--overload` /
  `--no-override` semantics, and explicit precedence rules (env vars >
  overlay > `.env`).
- Works with **every language**, not just Node — `dotenvx run -- python
  app.py`, `dotenvx run -- ./my-rust-binary`, `dotenvx run -- cargo run`
  all pick up the same `.env`, so a polyglot repo has one secret-handling
  story instead of one per language.
- **Pro features** (radar, hub, precommit) live behind a paid tier; the
  core CLI (load, encrypt, decrypt, run, set, get, ls, keypair, ext) is
  fully open-source under BSD-3-Clause and the binary downloads from the
  GitHub releases are not gated.

## Example

```sh
# Replace `node -r dotenv/config index.js` with this
dotenvx run -- node index.js

# Encrypt the .env so it's safe to commit
dotenvx encrypt
# Now .env has `encrypted:...` values; .env.keys has the private key
# (add .env.keys to .gitignore; commit .env)

# Decrypt at runtime in CI / prod (the only thing CI needs)
DOTENV_PRIVATE_KEY="$(cat /run/secrets/dotenv_key)" dotenvx run -- ./server

# Per-environment file: .env.production loaded when NODE_ENV=production
dotenvx run --env-file=.env.production -- npm start

# Read / write a single key from scripts
dotenvx get DATABASE_URL
dotenvx set NEW_FLAG true --encrypt
```

## When to use it

Reach for `dotenvx` when a project's `.env`-shaped secret-handling has
outgrown plain `dotenv` but you do not want to adopt a full secret
manager (AWS Secrets Manager, HashiCorp Vault, Doppler, 1Password
Connect) for what is fundamentally a small list of API keys per
environment:

- Solo dev / small team where the deployment story is "one Linux box
  running `systemd` units" or "Render / Fly.io / a docker image" and
  the operational overhead of a secret manager is the wrong tradeoff.
- Polyglot repos (Node + Python + Rust + a shell script) that want one
  secret-handling story across all the runtimes — every language reads
  the same `.env`, encrypted or plain, via `dotenvx run --`.
- Open-source repos where you want the `.env.example` *and* the actual
  `.env` (encrypted) committed so a contributor can run the project
  end-to-end after the maintainer hands them a private key over a
  separate channel — no separate "ask Slack for credentials" loop.

Skip when the team already has a real secret manager wired in (use it),
when the secrets are large blobs / certificates / per-user OAuth tokens
(`.env`-shaped storage is the wrong fit), or when the deployment
substrate already injects env vars natively (Kubernetes Secrets +
`envFrom` is fine on its own). Pairs cleanly with [`age`](../age/) for
file-shaped (not env-shaped) secrets and with [`sops`](../sops/) when
the secret store wants to be the YAML / JSON config file itself rather
than `.env`.
