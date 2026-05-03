# mage

- **Repo:** https://github.com/magefile/mage
- **Version:** 1.17.2 (latest stable, 2026-04-23)
- **License:** Apache-2.0 ([LICENSE](https://github.com/magefile/mage/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install mage` · `go install github.com/magefile/mage@latest` · `pacman -S mage` (AUR) · prebuilt binaries from [GitHub releases](https://github.com/magefile/mage/releases) · `go run github.com/magefile/mage` for a hermetic build-time invocation without installing the binary

## What it does

`mage` is **Make for Go projects, written in Go**. Instead of a `Makefile` whose recipes are shell snippets glued together with tabs, you write `magefile.go` — a normal Go source file in package `main` with the `//go:build mage` build tag — and every exported function in it becomes a callable target. `mage build` runs `func Build()`; `mage test` runs `func Test()`; `mage release v1.2.3` runs `func Release(version string)` with the positional arg type-coerced from the CLI. Targets can take `context.Context` as the first parameter (so cancellation / timeouts propagate from `mage -t 30s`), can return `error` (which mage prints and exits 1 on), and can declare dependencies on other targets via `mg.Deps(Build, Lint)` / `mg.SerialDeps(...)` / `mg.CtxDeps(ctx, Test)` — mage memoizes each target invocation per process, so a diamond dependency graph runs each node exactly once. Targets can be grouped into namespaces by declaring a `type Build mg.Namespace` zero-value struct and hanging methods off it (`func (Build) Linux() error`, `func (Build) Darwin() error`) — `mage build:linux` and `mage build:darwin` then route to the methods. The `sh` helper package (`github.com/magefile/mage/sh`) gives you `sh.Run("go", "build", "./...")`, `sh.Output(...)`, `sh.RunV(...)` (verbose), and `sh.RunWith(env, ...)` so you don't have to wrap every shell call in `os/exec` boilerplate; the `target` package gives you `target.Path(out, deps...)` (rebuild-if-newer) and `target.Glob(out, "src/**/*.go")` (Make-style staleness checks). The `mage` binary itself is tiny (~5 MB) and operates by transparently compiling `magefile.go` to a cached binary on first run, then re-using the binary on subsequent runs until the source mtime changes — so iteration is fast even though you're "compiling your build script". `mage -compile ./bin/build` produces a self-contained build binary you can ship to CI runners that don't have mage installed; `mage -l` lists targets with their first-line doc comments; `mage -h Target` prints the long-form doc comment for a specific target. v1.17.x added improved Windows path handling and tightened the `-compile` flag's reproducibility story.

## When to pick it / when not to

Pick `mage` when (a) **the project is already Go** and the `make build && make test && make release` shell soup has grown to the point where a missing tab or an unquoted `$VAR` is causing real bugs; (b) you want **dependency-graph evaluation, parallel target execution, and memoization** without writing it yourself in shell; (c) you want build steps to be **debuggable in your IDE** — `magefile.go` is a normal Go file, you can set breakpoints, run tests against build helpers, and the same lint/format/vet tooling applies; (d) you want **cross-platform parity** out of the box — a magefile that calls `sh.Run("git", "rev-parse", "HEAD")` works identically on macOS, Linux, and Windows whereas a Makefile rapidly accumulates `ifeq ($(OS),Windows_NT)` branches; (e) you want **build steps to receive typed args** — `func Release(version string, dryRun bool)` is straightforwardly callable as `mage release v1.2.3 true`. Common production shapes: replacing a 400-line Makefile in a monorepo Go service with a `magefile.go` that has `Build`, `Test`, `Lint`, `Release`, `Docker:Build`, `Docker:Push`, and `Migrate:Up`/`Down` namespaces; using `mage -compile` in CI to produce a stable build binary that is then `git`-pinned; embedding mage in a developer-facing CLI where every project subcommand is a magefile target. Pair with [`goreleaser`](../goreleaser/) for the actual cross-compile + tarball + GitHub-release step (mage invokes goreleaser; you don't reimplement it). Pair with [`gotestsum`](../gotestsum/) inside `func Test()` for nicer test output. Pair with [`golangci-lint`](../golangci-lint/) inside `func Lint()`. Pair with [`task`](../task/) when the project is *not* Go and you want a YAML-defined task runner with similar ergonomics.

Skip mage when (a) the project is **not Go and has no Go developers** — bringing a Go toolchain in just to run the build script is overkill; use [`task`](../task/) (Taskfile.yml), [`just`](../just/) (Justfile), or plain Make instead; (b) the build is **trivially three shell lines** — `make build` against a four-line Makefile is fine, the mage cost is the magefile.go boilerplate; (c) the team is allergic to Go and `magefile.go` would become a single-maintainer artifact; (d) you need **a single binary that bundles task definitions in a config file** — mage is binary + sources, not pure config; just/task fit "one binary, one declarative file" better; (e) you need **Windows-cmd compatibility for users who don't have a Go toolchain** — even with `-compile`, the developer ergonomics assume Go is installed.

## Vs already cataloged

- **Vs [`task`](../task/) (Taskfile.dev):** the closest comparison and the most common alternative. `task` is a single Go binary that reads a YAML `Taskfile.yml`; tasks are shell snippets with declarative `cmds`/`deps`/`sources`/`generates`. `task` wins on (a) language-agnostic projects (no Go required to author), (b) declarative config that diffs cleanly, (c) faster onboarding for non-Go teams. `mage` wins on (a) Go projects that want type-checked, IDE-debuggable build code, (b) complex conditional logic that gets ugly in YAML, (c) calling out to Go libraries directly without `sh` wrapper indirection. Pick task for polyglot repos, mage for Go-native build pipelines.
- **Vs [`just`](../just/):** Justfiles are a Make-shaped DSL with much better ergonomics than Make (no tab vs space, parameters, recipe deps, OS-conditional recipes). Just is language-agnostic and the file is human-readable config. Mage gives you a real programming language at the cost of "your build script is now a Go file you have to compile". Pick just for small to medium projects in any language; pick mage when build logic exceeds what's pleasant in a shell-flavored DSL.
- **Vs [`mask`](../mask/):** mask uses a Markdown file with shell code blocks as the task definition — readable as documentation, executable as a CLI. It's closest to just/task in spirit. Mage is in a different category (compiled program vs config file).
- **Vs [`pkgx`](../pkgx/) / [`mise`](../mise/):** mise can serve as a primitive task runner via `[tasks]` sections in `mise.toml`, but its primary job is runtime version management. Use mise *alongside* mage to pin the Go version that compiles the magefile.
- **Vs [`earthly`](../earthly/):** earthly is a containerized build system with a Makefile-shaped DSL where every step runs in a BuildKit container — strong reproducibility, hermetic by design. Mage runs on the host. Pick earthly when reproducibility matters more than iteration speed; pick mage when you want fast inner-loop iteration on a developer laptop.
- **Vs [`dagger`](../dagger/):** dagger lets you write CI pipelines as code in Go/Python/TypeScript that execute inside BuildKit. Mage is much smaller in scope — it's a target runner, not a CI engine. They can compose: a magefile target can invoke a dagger pipeline.
- **Vs `make`:** mage explicitly positions itself as the Go-native Make alternative. If your team is comfortable with Make and the `Makefile` works, there is no urgent reason to migrate. Mage shines when the Makefile is fighting you.

## Caveats

- **Magefiles need a build tag.** The first line of every magefile must be `//go:build mage` (and historically also `// +build mage`) so `go build ./...` doesn't accidentally try to compile build code into the application. Forgetting the tag is the #1 first-time foot-gun — the file ends up in the production binary or a `go test` run.
- **Compilation cache lives in `$MAGEFILE_CACHE`** (default `~/.magefile`). On CI runners with ephemeral home directories, you pay the compile cost on every run unless you cache that directory or use `mage -compile` to pre-build. The compile takes ~1–3s for a typical magefile, which is fine for laptop use but adds up across many short CI jobs.
- **`mg.Deps` runs in parallel by default.** If your build steps mutate the working tree (e.g. two targets that both write `bin/`), use `mg.SerialDeps` or refactor into a single target — parallel mutation will produce intermittent failures.
- **Errors from `sh.Run` swallow command stderr unless you use `sh.RunV` or capture with `sh.Output`.** A common debugging frustration is "my build target failed but I can't see why" — switch to `sh.RunV` for verbose passthrough during development.
- **Args are positional and type-coerced.** `mage release v1.2.3 true` maps to `Release("v1.2.3", true)`. There is no flag parser — if you need rich flags, write `func Release()` to read from env vars or accept a single struct-shaped string and parse it yourself. Many projects expose mage targets as thin wrappers and put the real CLI surface in a separate `cmd/` binary.
- **Cross-platform shell quoting.** `sh.Run("bash", "-c", "...")` is not portable to Windows. Prefer the `sh` helpers' multi-arg form (`sh.Run("go", "build", "./...")`) and avoid shelling out to a specific shell unless you also fence the call with `runtime.GOOS`.
- **Go module hygiene matters.** A magefile that imports third-party packages adds those to your project's `go.mod`. Many teams put magefiles in a separate `magefiles/` directory with its own `go.mod` (mage supports this layout) so build deps don't leak into the production module graph.
- Apache-2.0 ([LICENSE](https://github.com/magefile/mage/blob/master/LICENSE)) — permissive; safe to vendor into commercial projects and to invoke from CI without licensing concerns.

## Example invocations

```bash
# Install
brew install mage                              # macOS
go install github.com/magefile/mage@latest     # any platform with Go
# or one-shot, no install:
go run github.com/magefile/mage build

# Discover targets
mage -l                                        # list all targets with one-line docs
mage -h release                                # long-form doc for a specific target

# Run targets
mage build                                     # runs func Build()
mage test                                      # runs func Test()
mage docker:build                              # runs func (Docker) Build() in namespace
mage release v1.2.3                            # runs func Release(version string)

# Verbose, with timing
mage -v -t 60s build                           # -v = stream sh output, -t = timeout

# Force re-run (bypass mtime cache)
mage -f build

# Compile a portable, no-Go-required build binary for CI
mage -compile ./bin/mage-build
./bin/mage-build test                          # runs the same targets without Go installed

# Initialize a new magefile in the current directory
mage -init

# Minimal magefile.go to copy into a project (illustrative)
# //go:build mage
#
# package main
#
# import (
#   "github.com/magefile/mage/mg"
#   "github.com/magefile/mage/sh"
# )
#
# // Build compiles the binary into ./bin/app.
# func Build() error {
#   return sh.Run("go", "build", "-o", "bin/app", "./cmd/app")
# }
#
# // Test runs the unit tests.
# func Test() error {
#   return sh.RunV("go", "test", "-race", "./...")
# }
#
# // Release builds, tests, and publishes a tagged release.
# func Release(version string) error {
#   mg.SerialDeps(Build, Test)
#   return sh.Run("goreleaser", "release", "--clean")
# }
```
