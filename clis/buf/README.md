# buf

> **The best way of working with Protocol Buffers.** A single
> Go binary that owns the `.proto` lifecycle: lint
> (`buf lint`), breaking-change detection
> (`buf breaking --against '.git#branch=main'`), generation
> (`buf generate`, replacing `protoc` + a forest of plugins),
> dependency management via the Buf Schema Registry
> (`buf dep update`), and runtime invocation
> (`buf curl grpc+https://api.example.com/svc/Method`). Pinned
> to **v1.69.0**
> ([LICENSE](https://github.com/bufbuild/buf/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/bufbuild/buf>

## TL;DR

`buf` is the right answer when a project has more than three
`.proto` files and the `protoc` invocation in the `Makefile`
has started growing flag-by-flag. It replaces the `protoc` +
`protoc-gen-go` + `protoc-gen-go-grpc` + `protoc-gen-grpc-gateway`
+ `protoc-gen-validate` + `--proto_path=...:.../google:.../protovalidate`
spaghetti with one `buf.yaml` (the module: lint config,
breaking-change config, dependency list) plus one
`buf.gen.yaml` (the generation pipeline: which plugins, which
output paths). Plugins are referenced by name (`remote:
buf.build/protocolbuffers/go`) and resolved against the Buf
Schema Registry, so contributors do not need to install the
right `protoc-gen-*` versions on `$PATH`. `buf lint` enforces
the Google API Design Guide style by default; `buf breaking`
diffs the current schema against any git ref and fails CI if
a wire-incompatible change snuck through. For an LLM-CLI /
agent workflow, it is the deterministic, single-binary
contract for "this `.proto` change is safe to merge".

## Install

```bash
# Homebrew
brew install bufbuild/buf/buf

# Go
go install github.com/bufbuild/buf/cmd/buf@latest

# npm (wraps the prebuilt binary)
npm install -g @bufbuild/buf

# from a release tarball
curl -sSL "https://github.com/bufbuild/buf/releases/download/v1.69.0/buf-Darwin-arm64.tar.gz" | \
  tar -xz -C /tmp && sudo mv /tmp/buf/bin/buf /usr/local/bin/buf

# Linux package managers
# Arch (AUR): yay -S buf
# Nix:        nix-env -iA nixpkgs.buf

# Docker
docker run --rm -v "$PWD":/workspace -w /workspace bufbuild/buf:1.69.0 lint

# verify
buf --version    # 1.69.0
```

Initial setup is two files:

```yaml
# buf.yaml — the module
version: v2
modules:
  - path: proto
deps:
  - buf.build/googleapis/googleapis
  - buf.build/bufbuild/protovalidate
lint:
  use:
    - STANDARD
breaking:
  use:
    - WIRE
```

```yaml
# buf.gen.yaml — the codegen pipeline
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go:v1.36.5
    out: gen/go
    opt: paths=source_relative
  - remote: buf.build/grpc/go:v1.5.1
    out: gen/go
    opt: paths=source_relative
```

## License

Apache-2.0 — see
[LICENSE](https://github.com/bufbuild/buf/blob/main/LICENSE).
Permissive, requires preserving the copyright header in
redistributed source.

## One Concrete Example

```bash
# 1. lint the schema (Google API style by default)
buf lint
# proto/example/v1/foo.proto:14:1:Service name "fooSvc" should be PascalCase, e.g. "FooSvc".

# 2. fail CI if any change to a .proto on this branch is wire-breaking
#    vs main on origin
buf breaking --against '.git#branch=main,subdir=proto'

# 3. generate Go + gRPC + gRPC-Gateway from buf.gen.yaml
#    (no local protoc-gen-* binaries needed; plugins resolve remotely)
buf generate

# 4. push the schema to the Buf Schema Registry as a versioned module
buf push --tag "$(git rev-parse HEAD)"

# 5. invoke a live gRPC server from the schema, no .proto file in hand
#    (descriptors fetched from the BSR)
buf curl --schema buf.build/myorg/api \
  --data '{"id":"123"}' \
  grpc+https://api.example.com/myorg.api.v1.FooService/GetFoo

# 6. format every .proto in the tree (gofmt for protobuf)
buf format -w

# 7. validate that a JSON / YAML payload conforms to a message type
echo '{"id":"123","name":"x"}' | \
  buf convert proto/example/v1/foo.proto --type example.v1.Foo --from -

# 8. depend on googleapis without vendoring the source tree
buf dep update     # writes buf.lock with pinned digests
```

## Niche It Fills

**One binary, one config, full Protobuf lifecycle.** A typical
gRPC project ends up shelling out to `protoc` with five
`-I` paths, four `--*_out=...` flags, three plugin binaries
that have to be the right version on `$PATH`, an ad-hoc
`Makefile` rule for linting (or none), and a manual code
review for "is this `.proto` change wire-compatible?". `buf`
collapses every step into named subcommands with versioned
plugin references, a typed config file, and a default lint
ruleset that matches the Google API Design Guide. The Buf
Schema Registry on top adds versioned, dependency-managed
schemas (so `googleapis` is `buf.build/googleapis/googleapis`,
not a git submodule).

## Why use it

Three things `buf` does that `protoc` + Makefiles do not, that
pay back the switching cost on the first wire-breaking PR you
catch in CI:

1. **Wire-compatibility checking as a first-class command.**
   `buf breaking --against '.git#branch=main'` diffs the
   current `.proto` set against any git ref and fails on
   wire-incompatible changes (renamed fields keeping the
   same number, changed field types, removed required
   fields, renumbered enum values). No more "we shipped a
   schema break and noticed when the mobile client crashed";
   it is a CI gate.
2. **Remote plugins resolve at run time, version-pinned.**
   `remote: buf.build/protocolbuffers/go:v1.36.5` in
   `buf.gen.yaml` means CI does not need `protoc-gen-go` on
   `$PATH`; `buf generate` calls a hosted runner with that
   exact plugin version. Contributors with no Go toolchain
   can still regenerate. The version is in the file, in the
   repo, in the diff.
3. **Buf Schema Registry replaces git-submoduling.** Depending
   on `googleapis` is `deps: [buf.build/googleapis/googleapis]`
   in `buf.yaml`; `buf dep update` resolves and pins to a
   `buf.lock` digest, exactly like `go.sum` / `Cargo.lock`.
   No more `git submodule update --init --recursive` rituals.

For an LLM-CLI / agent workflow, `buf lint` + `buf breaking`
turn `.proto` review into a machine-checked gate the agent
can run before proposing a diff, and `buf curl` lets the
agent talk to a live gRPC server using only the schema name
in the BSR, no hand-built client.

## Vs Already Cataloged

- **Vs [`grpcurl`](../grpcurl/) (cataloged):** overlapping on
  the "invoke a live gRPC server from the CLI" job. `grpcurl`
  is the long-standing tool for that one job, schema fetched
  via gRPC reflection or a local `.proto`. `buf curl` is
  newer, integrates with the Buf Schema Registry (so the
  schema travels with the module name, not the server's
  reflection endpoint), and shares the `buf` config /
  authentication. Pick `grpcurl` for one-off probes against
  arbitrary servers; pick `buf curl` when the schema lives
  in the BSR and you want one binary across lint / generate
  / invoke.
- **Vs `protoc` (not cataloged):** the upstream reference
  compiler. `buf` does not embed `protoc` — it has its own
  parser — but it can call any `protoc-gen-*` plugin, so
  every plugin in the protoc ecosystem still works. Pick
  `protoc` for tiny schemas, scripts that call the C++
  parser, or as a sanity check; pick `buf` for any project
  big enough to have a `Makefile` rule.
- **Vs `prost` / `tonic-build` (not cataloged, Rust-side):**
  orthogonal at the language level. Rust projects typically
  generate via `prost`-based build scripts; `buf` can also
  drive Rust generation via the `community/neoeinstein-prost`
  remote plugin. The lint / breaking-change story is the
  selling point even if the generation step stays in
  `build.rs`.
- **Vs [`grpcui`](../grpcui/) — not cataloged here.** That is
  the interactive web UI peer to `grpcurl`. `buf` has no UI
  side; `buf curl` is the CLI-shaped invocation tool.

## Caveats

- **Two config files (`buf.yaml` + `buf.gen.yaml`) and a lock
  file (`buf.lock`).** Surface area is small but you have to
  remember which knob is in which file: module identity / lint
  / breaking / deps in `buf.yaml`, codegen plugins / out paths
  in `buf.gen.yaml`. The v2 config schema (current default)
  consolidates both into `version: v2`; older repos may still
  have v1 split files.
- **Remote plugins require network at generate time.** The BSR
  hosts the plugin runners; `buf generate` calls out to
  `buf.build` unless you switch a plugin to a `local:` /
  binary reference. For air-gapped builds, vendor plugins as
  local binaries (`local: ["protoc-gen-go"]`) and accept that
  contributors need them on `$PATH` again.
- **Default lint ruleset is opinionated (Google API Design
  Guide).** `STANDARD` enforces PascalCase services, `v1`
  package suffixes, required `_request` / `_response` message
  shapes. For brownfield `.proto` trees, expect to either
  ratchet (`ignore_only:` for legacy files) or relax the rule
  set (`use: [BASIC]`). The lint config block is where every
  retrofit project spends its first afternoon.
- **`buf push` puts the schema on `buf.build` by default.**
  For private modules either pay for BSR private hosting,
  self-host the registry, or skip `buf push` entirely and
  treat `buf` as a pure local tool. The lint / breaking /
  generate flows all work without ever pushing.
- **No code-generation cache.** `buf generate` regenerates
  every output every time; for huge schemas this is
  noticeable (seconds, not minutes, but not free). Use the
  `--include-imports=false --include-wkt=false` flags to
  trim, and pair with a build-system cache (`make` /
  `bazel`) that gates the call on `.proto` mtimes.
