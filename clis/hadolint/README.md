# hadolint

> **A Dockerfile linter** — parses your `Dockerfile`
> into an AST and runs ShellCheck against every
> `RUN` line, catching both Docker-specific
> antipatterns and shell bugs in one pass. Pinned
> to **v2.14.0**
> ([LICENSE](https://github.com/hadolint/hadolint/blob/master/LICENSE),
> GPL-3.0).

Source: <https://github.com/hadolint/hadolint>

## TL;DR

`hadolint` is the "stop me from shipping a 1.4 GB
container that runs as root" tool. Point it at a
`Dockerfile` and it walks the parsed instructions
applying ~80 named rules (`DL3008` "pin apt-get
versions", `DL3025` "use JSON form of CMD",
`DL4006` "set `SHELL` for pipefail", etc.), then
hands every `RUN` body to **ShellCheck** so you
also catch quoting bugs, unused variables, and
`cd $foo && rm -rf` foot-guns inside the image
build itself. Output is formatted as
human-readable, JSON, GCC-style, GitLab CI
codeclimate, SARIF, or Checkstyle, so it slots
into any pipeline. Per-rule severity, ignore
lists, and trusted-registry allowlists live in a
small `.hadolint.yaml`, and you can `# hadolint
ignore=DL3008` inline when you really do want to
skip a rule for one line.

## Install

```bash
# Homebrew
brew install hadolint

# Docker (no install needed)
docker run --rm -i hadolint/hadolint < Dockerfile

# prebuilt binaries (Linux / macOS / Windows)
# https://github.com/hadolint/hadolint/releases

# verify
hadolint --version    # Haskell Dockerfile Linter 2.14.0-no-git
```

## Basic usage

```bash
# lint a single Dockerfile
hadolint Dockerfile

# lint everything matching a glob
hadolint Dockerfile.* services/*/Dockerfile

# JSON output for CI
hadolint -f json Dockerfile

# SARIF for code-scanning dashboards
hadolint -f sarif Dockerfile > hadolint.sarif

# project config
cat > .hadolint.yaml <<'EOF'
ignored:
  - DL3008   # we pin via a separate manifest
trustedRegistries:
  - ghcr.io
  - docker.io
EOF
```

## When to choose

- **You build container images and want a
  pre-commit gate** — hadolint is fast (single
  static binary) and produces machine-readable
  output, perfect for `pre-commit` /
  `lefthook` / CI.
- **You want shell linting inside `RUN` steps
  without a separate pass** — it bundles
  ShellCheck so `apt-get update && apt-get
  install -y foo` gets the same scrutiny as a
  standalone `.sh` script.
- **You enforce org-wide image policy** —
  `.hadolint.yaml` lets you mark certain rules as
  errors (fail CI) vs warnings, and lock down
  which registries `FROM` may pull from.

## When NOT to choose

- **You want runtime image scanning for CVEs** —
  use [`trivy`](../trivy/) or
  [`grype`](../grype/); hadolint is a static
  Dockerfile linter, not a vulnerability scanner.
- **You write OCI manifests by hand or use
  Buildah scripts** — hadolint only understands
  the `Dockerfile` DSL.
- **You want auto-fix** — hadolint reports
  issues but does not rewrite Dockerfiles; pair
  it with a formatter or fix manually.

## Why it fits the zoo

Slots into the "static analysis for a specific
file format" cluster alongside
[`shellcheck`](../shellcheck/),
[`yamllint`](../yamllint/), and
[`actionlint`](../actionlint/). hadolint's
specific gap is **Dockerfile-aware linting that
reuses ShellCheck for the embedded shell** — no
other tool covers both the Docker DSL and the
shell semantics inside `RUN` in one walk.

## Upstream pointers

- Repo: <https://github.com/hadolint/hadolint>
- Release notes: <https://github.com/hadolint/hadolint/releases>
- License: [GPL-3.0](https://github.com/hadolint/hadolint/blob/master/LICENSE) (`LICENSE`)
- Maintainer: [@hadolint](https://github.com/hadolint) org
