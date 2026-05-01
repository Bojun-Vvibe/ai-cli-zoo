# yamlfmt

## What it does
Opinionated YAML formatter from Google: parses every YAML document in a path glob with a real YAML 1.2 parser, normalises indentation / line length / quote style / trailing-newline / empty-line policy / map-key alignment, and writes the canonical form back in place. Supports a `basic` formatter (the default, configurable via `.yamlfmt`) and as of v0.21.0 a `kyaml` formatter that emits the Kubernetes-project KYAML dialect. Multi-document streams (`---`-separated) are formatted document-by-document so a single bad document does not poison the rest of the file.

## Why it's interesting
The "yaml.dump" / `yq -i` round-trip rewrites quoting and key order arbitrarily and produces churn-heavy diffs; `prettier --parser yaml` is a JS dependency and does not understand multi-document streams or merge-key (`<<: *anchor`) the way the Go parser does. `yamlfmt` is a single static Go binary, has a real config file (so a monorepo can pin `line_length: 120`, `retain_line_breaks: true`, `scan_folded_as_literal: true` once), and emits a `--lint` mode that exits non-zero on diff for CI gating without rewriting in place. The `kyaml` mode is the only off-the-shelf formatter that targets the new Kubernetes encoding.

## Niche category
Formatter — YAML pretty-printer with config + lint mode

## Repo
https://github.com/google/yamlfmt

## Version pinned
`v0.21.0`

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
brew install yamlfmt
# or
go install github.com/google/yamlfmt/cmd/yamlfmt@v0.21.0
# or download yamlfmt_0.21.0_Darwin_arm64.tar.gz from the release
```

## Usage examples
```sh
# Format every yaml file under cwd in place
yamlfmt .

# Lint-only mode for CI (exit 1 on diff, no rewrite)
yamlfmt -lint .

# Print the diff a rewrite would produce, without applying
yamlfmt -dry .

# Use the new KYAML formatter for Kubernetes manifests
yamlfmt -formatter type=kyaml manifests/

# Pin policy in .yamlfmt at repo root, then a bare `yamlfmt .` is reproducible
cat > .yamlfmt <<'YAML'
formatter:
  type: basic
  retain_line_breaks: true
  scan_folded_as_literal: true
  max_line_length: 120
YAML
```
