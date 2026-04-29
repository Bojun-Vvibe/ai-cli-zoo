# ripsecrets

> **A pre-commit-grade secret scanner in Rust** — built to be the
> fastest "stop me from committing an API key" check that can run
> inside a `pre-commit` hook on every staged diff without anyone
> noticing the latency. Pinned to **v0.1.11** (homebrew
> `ripsecrets`),
> [LICENSE](https://github.com/sirwart/ripsecrets/blob/master/LICENSE),
> MIT.

Source: <https://github.com/sirwart/ripsecrets>

## TL;DR

`ripsecrets` is the "scan a directory or a list of files for
hard-coded secrets, exit non-zero if any are found" tool you point
your `pre-commit` config at. It is intentionally small and fast:
one Rust binary, no Python interpreter, no Docker, no daemon —
designed so that the hook runs in milliseconds on a normal staged
diff and seconds on a full-repo audit. The detection strategy is
two-layer: a regex pass for known high-entropy patterns
(AWS keys, GitHub tokens, Stripe keys, JWTs, generic
`api_key=...` / `secret=...` / `password=...` assignments,
private-key PEM headers) plus an entropy filter on candidate
substrings to suppress matches inside long base64 blobs that
aren't actually secrets. False positives are managed via inline
`# pragma: allowlist secret` comments and a project-level
`.secretsignore` file that uses gitignore-style patterns. The
binary is wired to be invoked either as `ripsecrets <paths>`
(audit mode, walks recursively, respects `.gitignore`) or as
`ripsecrets --strict-ignore $(git diff --cached --name-only)`
inside a hook (only check what's about to be committed). It
publishes its own `.pre-commit-hooks.yaml`, so `repo:
https://github.com/sirwart/ripsecrets / rev: v0.1.11 / hook id:
ripsecrets` is the entire `pre-commit` integration.

## Install

```bash
# Homebrew (macOS / Linux)
brew install ripsecrets

# Cargo
cargo install ripsecrets --locked

# Nix
nix-env -iA nixpkgs.ripsecrets

# from a release binary
curl -L https://github.com/sirwart/ripsecrets/releases/download/v0.1.11/ripsecrets-x86_64-unknown-linux-musl.tar.gz | tar xz
sudo install ripsecrets /usr/local/bin/

# verify
ripsecrets --version    # ripsecrets 0.1.11
```

## What it does

- Walk a path (or a list of staged files) and exit non-zero on
  any line that looks like a hard-coded secret.
- Pattern + entropy detector covering AWS, GitHub, Stripe, JWT,
  PEM private keys, generic `api_key=`/`secret=` assignments,
  and high-entropy hex/base64 blobs.
- Respects `.gitignore` and a separate `.secretsignore` for
  per-project allow rules.
- Inline `# pragma: allowlist secret` to suppress a single line.
- Ships a `.pre-commit-hooks.yaml` for one-line `pre-commit`
  framework integration.
- Static binary, no runtime dependency — usable from minimal CI
  containers, Alpine, scratch images.

## pew-related use cases

- **Local pre-commit gate**: drop into `.pre-commit-config.yaml`
  so `git commit` blocks the moment a developer pastes an AWS
  key or `OPENAI_API_KEY=sk-...` into a config file.
- **CI safety net** in a `lint` job that runs `ripsecrets .`
  against the full tree on every PR, so a hook that was bypassed
  with `--no-verify` still gets caught before merge.
- **One-shot audit of an inherited repo** before publishing it:
  `ripsecrets .` walks the whole tree (including history-checked
  files) and prints every suspect line with a path:line locator.
- **Sanitization step before sharing a repro repo with a vendor**:
  scan an exported tarball for forgotten tokens before zipping it
  up.
- **Sanity check for AI-generated code** (paired with
  [code2prompt](../code2prompt/), [repomix](../repomix/), or any
  agent that writes files): run `ripsecrets <output-dir>` after
  a code-generation step to confirm the model did not echo a
  prompt-embedded secret back into the workspace.
