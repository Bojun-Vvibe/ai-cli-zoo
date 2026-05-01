# addlicense

## What it does
A **license-header inserter** written in Go: walks one or more
directory trees, detects each file's language by extension, prepends
a comment-syntax-correct copyright/license header if missing, and
optionally verifies in `--check` mode (CI-friendly exit code) that
every source file already carries one. Out of the box it knows
roughly 40 file types (Go, Rust, Python, JS/TS, Java, Kotlin, C / C++
/ headers, Shell, Bash, Zsh, Ruby, PHP, HTML, CSS, Less, GraphQL,
Lua, Elixir, Julia, Raku, Scheme, Vim, AWK, Powershell `.ps1` /
`.psm1`, Cython, BUILD / BUILD.bazel / `.buck2` files, Nix scripts,
TOML, YAML, Dockerfile, Makefile, Markdown, `.j2` Jinja, Rackup
`.ru`, Gradle) and picks the right comment style (`//`, `#`,
`<!-- -->`, `(* *)`, `;;`, etc.) per language. The header text is
either built-in (Apache-2.0, MIT, BSD, MPL-2.0, ISC, AGPL, …
selected via `-l <spdx>`) or supplied as a custom template file
(`-f LICENSE.tmpl`) that interpolates `{{.Year}}` and `{{.Holder}}`.
Skip rules accept gitignore-style globs (`-ignore vendor/**` /
`-ignore '**/*.pb.go'`) and a `.licenserc.toml` config can declare
defaults so contributors don't need to remember flags. Pre-commit
integration (`pre-commit-hooks` entry point) is built in as of
v1.2.0.

## Why it's interesting
Different shape from `licensecheck` / `askalono` / `ScanCode`
(license *detection* against a corpus — different problem: identify
what license a third-party file is under), from `reuse-tool` (REUSE
spec compliance — also great, but Python, dual-tracks SPDX headers
+ `.reuse/dep5` + `LICENSES/` directory; heavier ceremony for
projects that just want a header on every file), from `licensor`
(template generator for the top-level `LICENSE` file only, doesn't
touch source headers), and from custom `sed` / `awk` one-shots
(brittle on multi-language repos with shebangs, BOMs, build-tool DSL
files). addlicense is the *one Apache-2.0 Go binary, walks the
tree, knows every comment syntax, idempotent, CI-checkable* shape:
pick it specifically for new repos that need to bring 100 % of
sources into compliance before open-sourcing, for monorepos with
mixed languages where a single tool needs to handle Go + TS + Bazel
BUILD files in one pass, and for `pre-commit` hooks that should
fail PRs missing headers. Do **not** pick it for SPDX REUSE
compliance (use `reuse`), for license *scanning* of dependencies
(use `scancode-toolkit` / `licensee` / `askalono`), or for the
top-level `LICENSE` file itself (use `licensor` / `gh repo create
--license`).

## Niche category
License-header automation — Go single-binary that inserts and
verifies SPDX-style copyright headers across multi-language repos,
with built-in pre-commit and CI gating.

## Repo
https://github.com/google/addlicense

## Version pinned
`v1.2.0` (latest tagged release as of 2026-05-02, published
2025-08-13)

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Go install (any platform with a Go toolchain)
go install github.com/google/addlicense@v1.2.0

# Homebrew (community tap)
brew install addlicense

# Pre-built binaries for darwin / linux / windows
# https://github.com/google/addlicense/releases/tag/v1.2.0

# Container
docker run --rm -v "$PWD":/src ghcr.io/google/addlicense:v1.2.0 \
  -l apache -c "Acme Corp" -y 2026 -check ./...
```

## Usage examples
```sh
# Add Apache-2.0 headers (with copyright holder + year) to every source file
addlicense -l apache -c "Acme Corp" -y 2026 ./...

# CI gate: fail the build if any tracked file is missing a header
addlicense -check -l apache -c "Acme Corp" -y 2026 \
  -ignore '**/vendor/**' -ignore '**/*.pb.go' ./...

# Use a custom header template (multi-line, with company-specific notice)
addlicense -f .license-header.tmpl -ignore 'third_party/**' ./...

# Limit to specific extensions in a polyglot monorepo
addlicense -l mit -c "Jane Doe" \
  -ignore '**/*.json' -ignore '**/*.md' \
  ./services/ ./pkg/ ./scripts/

# Wire into pre-commit (.pre-commit-config.yaml)
# - repo: https://github.com/google/addlicense
#   rev: v1.2.0
#   hooks:
#     - id: addlicense
#       args: [-l, apache, -c, "Acme Corp", -y, "2026"]
```

## Date added
2026-05-02
