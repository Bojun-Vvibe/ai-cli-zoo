# task (Taskfile)

- **Repo:** https://github.com/go-task/task
- **Version:** v3.50.0 (latest stable, 2026-04)
- **License:** MIT ([LICENSE](https://github.com/go-task/task/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install go-task` · `go install github.com/go-task/task/v3/cmd/task@latest` · `npm i -g @go-task/cli` · binary releases on the GitHub release page

## What it does

`task` is a YAML-defined task runner — a `make` replacement for the polyglot
project that does not want to write recipe-shaped Makefiles or learn `just`'s
own DSL. The contract is a `Taskfile.yml` at the repo root declaring named
tasks with `cmds:` (a list of shell commands), `deps:` (other tasks to run
first, in parallel by default), `sources:` and `generates:` (file-glob
fingerprints for incremental skip-if-up-to-date — the `make` story without
the tab-vs-space DSL), `vars:` (per-task or top-level variables, with
`{{.VAR}}` Go-template interpolation), `env:`, `dotenv:` (auto-load
`.env`-style files), `preconditions:`, `status:`, `requires:`, `silent:`, and
`includes:` for splitting large Taskfiles across files / repos. `task --list`
prints documented tasks, `task --watch <name>` re-runs on file changes,
`task --parallel a b c` fans out, `--summary <name>` prints a task's full
spec, and `task --concurrency N` caps parallel deps. Cross-platform: the
shell-command runner is `mvdan/sh` (a Go reimpl of POSIX shell) so the same
`Taskfile.yml` runs on Linux, macOS, and Windows without `bash` installed —
which is the differentiator versus `make` for teams that include Windows devs.

## When to pick it / when not to

Pick `task` when you want a polyglot task runner with a YAML contract,
file-fingerprint incremental builds, and Windows-without-bash portability.
The sweet spot is "we have a Node frontend, a Go backend, a Python data
pipeline, a Terraform module, and a Docker compose stack and we want one
command surface (`task lint`, `task test`, `task deploy:staging`) that works
on every contributor's laptop and in CI". The `includes:` mechanism scales to
monorepos cleanly (one root Taskfile delegates to per-package Taskfiles).

Skip it when the project is single-language and the language already has a
canonical task runner (`npm scripts` for Node, `cargo` aliases for Rust,
`poetry` / `uv` scripts for Python, `mix` for Elixir) — adding a second
runner is friction with no payoff. Skip it for genuine build-system work
(complex DAG, sandboxed actions, remote cache, hermetic toolchains) where
[`bazel`](https://bazel.build/) / `buck2` / `nx` are the right layer — `task`
deliberately does not solve those. Pick [`just`](../just/) over `task` when
you prefer a smaller, more `make`-shaped DSL with no YAML and stronger
recipe-as-script ergonomics; pick `task` over `just` when YAML readability,
file-fingerprint incremental skip, and richer `includes:` matter more than
DSL minimalism. Pairs naturally with [`mprocs`](../mprocs/) (long-lived dev
processes the way `task` is short-lived recipes) and
[`watchexec`](../watchexec/) (`task --watch` already covers the common case
but `watchexec` handles non-Taskfile triggers).

## Example invocations

```bash
# Run the default task (or the one named "default")
task

# List documented tasks (anything with a `desc:` field)
task --list

# Run a specific task, passing CLI args after --
task test -- --run TestSomething

# Run two tasks in parallel
task --parallel lint test

# Watch source files and re-run on change
task --watch build

# Print everything task would do without executing (great for CI debugging)
task --dry deploy:staging

# Force re-run even when sources/generates fingerprint says up-to-date
task --force build
```

Minimal `Taskfile.yml` for reference:

```yaml
version: '3'

vars:
  BIN: ./bin/myapp

tasks:
  build:
    desc: Compile the binary
    sources: ['**/*.go']
    generates: ['{{.BIN}}']
    cmds:
      - go build -o {{.BIN}} ./cmd/myapp

  test:
    desc: Run the test suite
    cmds:
      - go test ./...

  ci:
    desc: Lint + test + build (parallel deps)
    deps: [test, build]
```
