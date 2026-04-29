# earthly

> **A repeatable build automation tool that combines Dockerfile syntax
> with Makefile semantics — `Earthfile` declares targets (`+build`,
> `+test`, `+lint`, `+deploy`) that run inside hermetic BuildKit
> containers with explicit input artifacts, output artifacts (`SAVE
> ARTIFACT` and `SAVE IMAGE`), and target dependencies (`BUILD +foo`,
> `FROM +base`, `COPY +artifact-target/file ./`), so the same `earthly
> +test` command runs identically on a developer laptop, in GitHub
> Actions, in GitLab CI, in Buildkite, and in Jenkins — no per-CI YAML
> rewrite, automatic cache reuse via BuildKit's content-addressable
> store, and a remote satellite mode (`earthly --org my-org sat
> select prod-sat`) that runs builds on a shared persistent BuildKit
> daemon for warm-cache CI.** Pinned to **v0.8.16**
> ([LICENSE](https://github.com/earthly/earthly/blob/main/LICENSE),
> MPL-2.0).

Source: <https://github.com/earthly/earthly>

## TL;DR

`earthly` is what to reach for when CI configs have grown into
3000-line YAML files with shell-out steps that work in CI but not
locally (or vice versa), when "reproduce the failing CI build on
my laptop" takes 40 minutes of installing Docker images and
fiddling with environment variables, and when the same build
logic is duplicated across `Dockerfile`, `Makefile`, GitHub
Actions YAML, and a `scripts/build.sh` that drift apart over
time. The fix is one `Earthfile` per repo declaring named build
targets in Dockerfile-like syntax (`FROM`, `COPY`, `RUN`,
`ARG`, `ENV`) plus Makefile-like target composition (`+target`
references, `BUILD`, `WAIT/END` for parallelism), all executed in
hermetic BuildKit containers with explicit declared inputs and
outputs. The same `earthly +test` runs the same way on your
laptop and in CI; cache is content-addressable and reused across
runs; remote satellites give CI the warm-cache benefit of a
long-running build server without each developer running their
own. The trade-off is one more tool to install (Earthly + Docker /
Podman) and a DSL to learn, but the win is collapsing four
build-system layers into one.

## Install

```bash
# Homebrew (macOS / Linux)
brew install earthly/earthly/earthly && earthly bootstrap

# Linux package managers
# Debian / Ubuntu (via apt repo): see
# https://earthly.dev/get-earthly
dnf install earthly                   # Fedora (third-party repo)
pacman -S earthly                     # Arch (AUR)

# Direct binary
curl -sSL https://github.com/earthly/earthly/releases/download/v0.8.16/earthly-linux-amd64 \
  -o /usr/local/bin/earthly && chmod +x /usr/local/bin/earthly && earthly bootstrap

# verify
earthly --version                     # earthly version v0.8.16 ...
```

`earthly bootstrap` installs shell completions and (on first run)
pulls the BuildKit daemon image. Requires Docker or Podman on the
host (BuildKit runs as a container).

## License

**MPL-2.0** — see
[LICENSE](https://github.com/earthly/earthly/blob/main/LICENSE).
File-level copyleft: modifications to MPL-licensed files must be
released under MPL when redistributed, but combining with code
under other licences is allowed (the obligation is per-file, not
per-program). Fine for using the binary in CI / dev workflows;
matters only if you fork and redistribute modified Earthly
source. Earthly Inc. also offers a hosted "Cloud" / "Satellites"
service under separate commercial terms; the OSS binary is
fully functional without it.

## One Concrete Example

```dockerfile
# Earthfile — declarative, repeatable build targets
VERSION 0.8
PROJECT my-org/my-service
FROM golang:1.22-bookworm
WORKDIR /src

# +deps: download go modules into a cached layer
deps:
    COPY go.mod go.sum ./
    RUN go mod download
    SAVE ARTIFACT go.mod
    SAVE ARTIFACT go.sum

# +build: compile binary, save as artifact
build:
    FROM +deps
    COPY . .
    RUN CGO_ENABLED=0 go build -o my-service ./cmd/my-service
    SAVE ARTIFACT my-service AS LOCAL ./bin/my-service

# +test: run go test, parallel with +lint
test:
    FROM +deps
    COPY . .
    RUN go test -race -coverprofile=coverage.out ./...
    SAVE ARTIFACT coverage.out AS LOCAL ./coverage.out

# +lint: golangci-lint, parallel with +test
lint:
    FROM golangci/golangci-lint:v1.59-alpine
    WORKDIR /src
    COPY . .
    RUN golangci-lint run --timeout 5m ./...

# +ci: run test + lint in parallel, then build
ci:
    BUILD +test
    BUILD +lint
    BUILD +build

# +docker: produce a runtime image from the build artifact
docker:
    FROM gcr.io/distroless/static-debian12
    COPY +build/my-service /my-service
    ENTRYPOINT ["/my-service"]
    SAVE IMAGE --push registry.example.com/my-service:latest

# +deploy: push image, then call deploy script
deploy:
    BUILD +docker
    LOCALLY
    RUN ./scripts/deploy.sh
```

```bash
# locally: run the same target CI runs
earthly +ci                           # parallel test + lint + build
earthly +test                         # only the test target
earthly +build                        # only build, save bin to ./bin/
earthly --push +deploy                # build, push, deploy

# clean cache
earthly prune --all

# remote satellite: warm-cache builds on a shared daemon
earthly sat select prod-sat
earthly +ci                           # uses the satellite's BuildKit
```

## Niche It Fills

**One declarative file that replaces `Dockerfile` + `Makefile` +
per-CI YAML + ad-hoc `scripts/build.sh`, runs identically on a
laptop and in any CI, and gives content-addressable cache reuse
across runs without per-CI configuration.** The shape is unique:
plain Dockerfiles can build images but can't express "run this
target, save this artifact to disk, depend on this other target's
output"; `Makefile`s express target dependencies but lack
hermeticity (your `make test` depends on what's installed on the
host); `bazel` / `pants` give hermeticity but require a full
adoption commitment with their own dep declaration system; per-CI
YAML files (`.github/workflows/`, `.gitlab-ci.yml`, `.buildkite/`)
encode build steps that don't run on a laptop. Earthfile collapses
all four into one file with Dockerfile syntax for "what to run"
and Makefile semantics for "how targets depend on each other,"
backed by BuildKit for hermetic execution and CAS-based cache.

## Why use it

Three things `earthly` does that the alternatives do not:

1. **Same command runs identically on a laptop and in any CI.**
   `earthly +test` works on your MacBook, in GitHub Actions, in
   GitLab CI, in Buildkite, and in Jenkins, with no per-CI YAML
   rewrite — the CI config is reduced to "install earthly, run
   `earthly +ci`, upload artifacts." Reproducing a CI failure
   locally is one command, not 40 minutes of installing the right
   Docker images and exporting the right env vars. This is the
   single biggest practical win and the main reason teams adopt
   Earthly.
2. **BuildKit content-addressable cache, automatically.** Every
   `RUN` / `COPY` / artifact is content-hashed; cache hits
   transparently across local + CI + remote satellite. Change one
   Go source file and only the targets downstream of that file
   re-run; everything else is a cache hit. No `cache: paths:`
   per-CI declaration, no manual `actions/cache` keys.
   Multi-platform builds (`--platform=linux/amd64,linux/arm64`)
   share the cache where layers are identical.
3. **Targets compose with explicit data flow.** `COPY
   +build/binary ./` declares "this target needs the `binary`
   output of the `+build` target" — the dependency graph is
   explicit and verifiable. `BUILD +test` runs a target for its
   side effects. `WAIT / END` blocks express "run these in
   parallel, wait for all to finish, then continue." `LOCALLY`
   targets escape the container for steps that genuinely must run
   on the host (deploys, secret-fetching). The DSL is small (~20
   keywords) and the Dockerfile-shaped surface is familiar.

For a polyglot monorepo with Go services + Python data jobs +
Node frontend + Terraform deploys + a release pipeline, one
`Earthfile` per package + a top-level `+ci` target that
`BUILD`s each one is the whole CI config. The GitHub Actions
workflow shrinks to "checkout, install earthly, run `earthly
+ci`."

## Vs Already Cataloged

- **Vs [`task`](../task/) / [`just`](../just/):** different layer.
  `task` and `just` are *task runners* — they orchestrate shell
  commands on the host, with no hermeticity. `earthly` is a *build
  system* — it runs targets inside hermetic containers with
  declared inputs / outputs and cache. They compose: a
  `Taskfile.yml` recipe can call `earthly +ci` for the heavy
  build, and use `task` for thin wrappers (`task deploy:staging`
  → `earthly --push +deploy --ENV=staging`). Pick `task` / `just`
  for ad-hoc developer commands; pick `earthly` when you also
  want CI-equivalence and cache reuse.
- **Vs [`devbox`](../devbox/) (if cataloged) / Nix:** orthogonal.
  Devbox / Nix give you hermetic *development environments* (the
  `go` and `node` on your laptop are the same versions as in CI);
  earthly gives you hermetic *build steps*. Common pairing: Nix /
  devbox for the dev shell that has `earthly` + `docker` + your
  language toolchains pinned, then `earthly +ci` inside that
  shell for the actual build.
- **Vs [`kustomize`](../kustomize/) / [`helm`](../helm/) (if
  cataloged):** different stage. Earthly *builds* the artifact (a
  container image, a binary, a tarball); kustomize / helm
  *deploy* the built artifact to Kubernetes. A typical pipeline
  is `earthly --push +docker` (build + push image) →
  `kustomize edit set image my-service=registry/my-service:$SHA`
  → `kubectl apply -k overlays/prod`.
- **Vs `Dockerfile` + `Makefile` + per-CI YAML (the status quo):**
  earthly is the explicit replacement. The migration is usually
  "translate your Dockerfile into a `+docker` target, translate
  your Makefile recipes into Earthfile targets, delete the per-CI
  YAML steps." After migration, the per-CI YAML is one job that
  installs earthly and runs `earthly +ci`.

## Caveats

- **One more tool to install.** Earthly requires Docker or Podman
  on the host (BuildKit runs as a container). For environments
  that already have Docker installed this is free; for ones that
  don't (e.g. an `apt`-only build server with no container
  runtime), it's a real install dependency. Earthly's recommended
  path is `earthly bootstrap` which sets it up.
- **MPL-2.0 file-level copyleft.** Using the binary in CI / dev
  is fine. Forking and modifying Earthly source files and
  redistributing the result requires releasing the modified files
  under MPL. Most teams never touch this; it matters only for
  vendor / fork scenarios.
- **Earthly Cloud / Satellites are a paid commercial service.** The
  OSS binary is fully functional without them — you can run
  Earthly entirely locally and in CI. Satellites are a
  convenience layer (a long-running BuildKit daemon shared across
  CI runs for warm cache); pricing applies if you adopt them.
- **DSL has its own syntax to learn.** While Dockerfile-like, the
  Earthfile keywords (`SAVE ARTIFACT`, `SAVE IMAGE`, `BUILD +x`,
  `FROM +x`, `COPY +x/file`, `WAIT/END`, `LOCALLY`, `PROJECT`,
  `VERSION`) are Earthly-specific. The learning curve is real
  but small (~a half-day for a developer comfortable with
  Dockerfiles); migration of existing CI is the bigger time sink.
- **Cache is per-machine unless you use satellites or remote
  cache.** A laptop and a fresh CI runner have separate caches.
  For CI cache reuse across runs you either use Earthly Cloud
  satellites (managed), self-host a remote BuildKit cache
  (`--remote-cache`), or accept cold-cache CI on every run.
- **Some Dockerfile features map awkwardly.** Multi-stage
  Dockerfiles translate to multiple Earthly targets, which is
  cleaner — but `--mount=type=cache` Dockerfile syntax has its
  own Earthly equivalent (`CACHE`) that needs to be learned.
  `HEREDOC` blocks in Dockerfiles work, but error messages can be
  less clear than running them as plain `RUN` commands.
- **Version 0.8.x is current stable; 0.9 is in development.** API
  is stable within 0.8.x; pin `VERSION 0.8` at the top of your
  Earthfile to lock the DSL contract. Major-version bumps (0.7
  → 0.8) have historically required minor Earthfile edits.
