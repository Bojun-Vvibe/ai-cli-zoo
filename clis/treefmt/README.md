# treefmt

## What it does
A polyglot formatter orchestrator: one config file (`treefmt.toml`) maps file globs to underlying formatters (`prettier`, `gofmt`, `rustfmt`, `black`, `ruff format`, `nixpkgs-fmt`, `shfmt`, `taplo`, `dprint`, `clang-format`, `stylua`, etc.), and `treefmt` discovers changed files, batches them by formatter, runs each formatter in parallel, and reports a unified diff + per-formatter timing summary. `treefmt --fail-on-change` makes it a CI gate; `treefmt --no-cache` forces re-run; `treefmt --formatters go,rust` scopes a run to a subset.

## Why it's interesting
Every polyglot repo eventually grows a `Makefile` target like `fmt:` that shells out to seven different formatters in serial, with no caching, no parallelism, and no consistent exit code. `treefmt` is the missing meta-formatter: a single Rust binary (the v2 rewrite by numtide) that hashes each file's content + formatter binary version into a per-project cache so unchanged files are skipped between runs (sub-second incremental on a 10k-file repo), runs each underlying formatter on the full batch of files it owns (one `prettier --write file1 file2 ... file500` call instead of 500 fork/exec cycles), and emits a single tabular summary `traversed N files | emitted N files for processing | formatted N files in Xs`. The config schema is declarative and version-controlled, so contributors don't need to know which formatter handles `.tf` vs `.nix` vs `.toml` — they run `treefmt` and the repo's `treefmt.toml` decides. Pairs naturally with `pre-commit` and Nix flakes (`nix fmt` delegates to treefmt when configured).

## Niche category
Polyglot formatter orchestrator — one config, parallel + cached invocation of N underlying formatters

## Repo
https://github.com/numtide/treefmt

## Version pinned
`v2.5.0`

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
# macOS / Linux via Homebrew
brew install treefmt

# Nix
nix profile install nixpkgs#treefmt

# Go install (requires Go 1.22+; v2 is Go-based, not Rust — earlier docs may say otherwise)
go install github.com/numtide/treefmt/v2@latest

# Pre-built binary
curl -L https://github.com/numtide/treefmt/releases/latest/download/treefmt_$(uname -s)_$(uname -m).tar.gz | tar xz
```

## Usage
```toml
# treefmt.toml at repo root
[formatter.go]
command = "gofmt"
options = ["-w"]
includes = ["*.go"]

[formatter.rust]
command = "rustfmt"
options = ["--edition", "2021"]
includes = ["*.rs"]

[formatter.prettier]
command = "prettier"
options = ["--write"]
includes = ["*.js", "*.ts", "*.tsx", "*.json", "*.md", "*.yaml", "*.yml"]

[formatter.shell]
command = "shfmt"
options = ["-w", "-i", "2"]
includes = ["*.sh"]
```

```sh
# Format the whole repo (uses cache, only re-formats changed files)
treefmt

# CI gate — exits non-zero if anything would be reformatted
treefmt --fail-on-change --no-cache

# Scope to a subset of formatters (e.g. CI matrix shard)
treefmt --formatters go,rust

# Initialize a starter config by detecting installed formatters
treefmt --init
```

## When to pick `treefmt` vs alternatives
- **vs `pre-commit`**: not exclusive — most repos run treefmt *under* pre-commit (`hooks: - id: treefmt`). pre-commit is the hook-runner / lifecycle layer; treefmt is the formatter-orchestration layer with shared caching across formatters. Use treefmt alone when you don't want a hook framework dependency.
- **vs [`dprint`](../dprint/)**: dprint is itself a multi-language formatter with WASM plugins and is the right pick when its plugin set covers your stack (TS/JS, Markdown, TOML, JSON, Dockerfile). Pick treefmt when you need to keep using language-native formatters (`gofmt`, `rustfmt`, `nixpkgs-fmt`, `clang-format`) that the team already standardizes on, *and* want one entry point across them.
- **vs a hand-rolled `Makefile fmt:` target**: treefmt gives you parallelism, content-hash caching, and a unified diff/exit-code contract for free, plus declarative config that survives team turnover.
- **vs each formatter directly (`gofmt ./... && rustfmt ...`)**: fine for single-language repos. The moment you have ≥3 languages and a "format everything" CI step, treefmt's cache + parallel batching pays for itself within one CI run.
