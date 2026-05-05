# bunster

> **Shell-script-to-static-binary compiler: feed it a `.sh`, get
> back a single self-contained executable with no `bash` runtime
> dependency on the target host.** Pinned to **v0.14.0**
> ([LICENSE](https://github.com/yassinebenaid/bunster/blob/master/LICENSE),
> BSD-3-Clause).

Source: <https://github.com/yassinebenaid/bunster>

## TL;DR

`bunster` is a Go-implemented compiler that parses a Bash script
into an AST, lowers it to Go source, and `go build`s the result
into a static binary. Unlike `shc` (which encrypts the script and
ships an interpreter wrapper) or `argbash` (which only generates
parser scaffolding), `bunster` actually translates control flow,
expansions, redirections, and pipelines to native Go — the
resulting binary does not require `/bin/bash` to exist on the
target machine. Useful when you need to ship a shell tool to a
distroless container, an Alpine image without `bash`, an
embedded device, or a corporate Windows workstation where end
users will never `chmod +x` a `.sh`.

## Install

```bash
# Go install (requires Go 1.22+)
go install github.com/yassinebenaid/bunster/cmd/bunster@v0.14.0

# Homebrew (third-party tap)
brew install yassinebenaid/tap/bunster

# Or download a release tarball
curl -sSfL https://github.com/yassinebenaid/bunster/releases/download/v0.14.0/bunster_Linux_x86_64.tar.gz \
  | tar -xz -C /usr/local/bin bunster

# verify
bunster version
```

## Examples

```bash
# Compile a single script to a native binary
bunster build deploy.sh -o deploy
file deploy
# -> deploy: ELF 64-bit LSB executable, x86-64, statically linked

# Cross-compile for a different target
GOOS=linux GOARCH=arm64 bunster build deploy.sh -o deploy-arm64

# Generate Go source without compiling (for inspection / vendoring)
bunster generate deploy.sh -o ./generated/
```

## When to choose it

Pick `bunster` when (a) you have a working shell script you do
**not** want to rewrite in Go / Rust, and (b) the deployment
target either lacks `bash` or you do not want to audit the
target's `bash` version compatibility (3.2 vs 5.x, GNU vs BusyBox
`ash`). Typical fits: bootstrap scripts inside distroless
containers, install scripts shipped to customers who run
heterogeneous Linux distros, internal tools handed to ops teams
that block interpreted-script execution on managed endpoints.

Skip it when the script is trivial enough to rewrite in Go
directly (the generated binary will be smaller and the source
more debuggable), when the script depends heavily on
`bash`-specific features that `bunster` does not yet implement
(check the project's compatibility matrix — `coproc`, certain
parameter-expansion edge cases, and `read -e` are still partial),
or when you actually need the script to be hot-editable on the
target host (a binary defeats the point).

## Vs adjacent tools

- **Vs `shc`:** `shc` wraps the original script inside a C
  binary that re-invokes `bash` at runtime. The target host
  still needs `bash`. `bunster` produces a true standalone
  binary.
- **Vs [`bashly`](../bashly/):** `bashly` *generates* a Bash
  script from a YAML CLI spec — the output is still a `.sh`
  that needs `bash` to run. They compose: write the CLI in
  `bashly`, then compile the generated script with `bunster`
  for distribution.
- **Vs rewriting in Go / Rust:** if the script is mostly glue
  around `kubectl` / `aws` / `git`, rewriting buys you almost
  nothing — the same external binaries still have to exist on
  the target. `bunster` keeps the shell-glue lifecycle (fast
  edits, `set -x` debugging) and only changes the
  *distribution* artifact.

## Caveats

- **Not 100% Bash-compatible.** The compatibility surface is
  growing but still incomplete. Test the generated binary
  against the same fixtures you used for the script — do not
  assume a green compile means semantic parity.
- **Binary size is real.** A trivial `echo hello` script
  compiles to a ~5 MB Go binary because the runtime has to
  carry a Bash semantics layer. Acceptable for `/usr/local/bin`,
  noticeable in a 50 MB container budget.
- **Compile-time only.** There is no `bunster eval` — once
  compiled, the script is frozen. If your workflow relies on
  editing the script on the target host, this tool is the
  wrong shape.
- **Active development; expect breaking changes.** The project
  is pre-1.0 (v0.14.0 at time of writing). Pin the compiler
  version in CI rather than tracking `@latest`, and re-test
  generated binaries when bumping versions.
