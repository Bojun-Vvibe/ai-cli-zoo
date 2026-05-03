# nixpacks

- **Repo:** https://github.com/railwayapp/nixpacks
- **Version:** v1.41.0 (latest stable, 2025-10-24)
- **License:** MIT ([LICENSE](https://github.com/railwayapp/nixpacks/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install nixpacks` · `curl -sSL https://nixpacks.com/install.sh | bash` · `cargo install nixpacks` · static binaries on the GitHub release page

## What it does

`nixpacks` is a Buildpacks-style build orchestrator that takes an
*app source directory* (no `Dockerfile`, no `Procfile`, no buildpack
declaration) and produces a runnable OCI image by detecting the
language / framework, resolving the right runtime + system packages
from the Nix package set, and emitting a multi-stage `Dockerfile`
that BuildKit then turns into the final image. The two key commands
are `nixpacks build ./app -t myapp:latest` (one-shot to a local
Docker image) and `nixpacks plan ./app` (print the JSON build plan
nixpacks *would* execute, useful for inspection / customisation /
CI gating).

The detection layer covers ~25 ecosystems out of the box — Node
(npm / pnpm / yarn / bun, with framework hints for Next / Remix /
Astro / SvelteKit), Python (pip / poetry / pdm / uv, with Django /
Flask / FastAPI hints), Go, Rust, Ruby (rails / sinatra), PHP, Java
(Maven / Gradle), Deno, Bun, Elixir, Crystal, Haskell, Zig, Swift,
Dart, Clojure, .NET, Cobol, Lunatic, Scala, Staticfile sites, and a
generic shell-script detector — and each ecosystem ships a curated
default install / build / start command set that you can override
with one of three escape hatches: `nixpacks.toml` at the repo root
(declarative override of phases + packages), `NIXPACKS_*` env vars
(`NIXPACKS_INSTALL_CMD`, `NIXPACKS_BUILD_CMD`, `NIXPACKS_START_CMD`,
`NIXPACKS_PKGS`, etc.), or `--build-cmd` / `--start-cmd` /
`--pkgs` flags directly on `nixpacks build`.

The reason it uses Nix under the hood is *deterministic system
packages* — when the build needs `ffmpeg` or `libpq` or `chromium`
or `tesseract`, nixpacks resolves them from a pinned `nixpkgs`
revision and adds them to the build image as content-addressed
derivations rather than `apt-get install`-ing whatever Debian
shipped this week. The resulting image is just a regular OCI image
(no Nix runtime needed in production), and the build is reproducible
because the nixpkgs pin lives in the nixpacks release.

## Pick over / pair with

- **Pick over a hand-rolled `Dockerfile`** for the *first* containerisation
  pass on any service in any of the ~25 supported ecosystems. The
  generated image is rarely the smallest possible (~150-500 MB depending
  on stack) but it is correct on day one with zero authoring; ship it,
  measure, and only hand-roll a Dockerfile when image size or build
  time becomes the bottleneck.
- **Pick over [`pack`](../pack/) (Cloud Native Buildpacks)** when you
  want one binary with no buildpack registry / builder image to pull
  and you want Nix-pinned system packages. Pick CNB / `pack` when the
  Paketo / Heroku buildpack ecosystem you already invest in is the
  asset, or when the rebase / Procfile / SBOM story matters more than
  raw simplicity.
- **Pick over [`ko`](../ko/) and [`apko`](../apko/)** when the source
  is a *full polyglot app* needing a build phase and system packages.
  ko/apko are sharper for "Go binary into a distroless image" and
  "declarative Alpine apk image" respectively; nixpacks is the broader
  catch-all for everything else.
- **Pick over [`buildah`](../buildah/) / [`docker build`](https://docs.docker.com/build/)**
  when you don't want to write the Dockerfile at all. nixpacks emits
  a Dockerfile under the hood that you can `nixpacks build --no-cache --print-dockerfile`
  and then maintain by hand once the project outgrows the auto-detect.
- **Pair with [`dive`](../dive/)** to inspect the resulting image
  layer-by-layer the first time you run nixpacks on a new project, so
  you understand what got pulled in.
- **Pair with [`cosign`](../cosign/)** to sign the produced image.
  nixpacks doesn't sign — it just produces an image; signing is a
  downstream concern.
- **Pair with [`flux`](../flux/) / [`argocd`](../argocd/)** as the
  consumer of the resulting image in a Kubernetes deploy pipeline.

## Caveats

- The output image size is *not* optimised for minimum bytes. nixpacks
  prioritises "build succeeds without you authoring a Dockerfile" over
  "smallest possible artifact"; expect 150-500 MB for typical
  Node/Python/Ruby apps. For minimum size, generate the Dockerfile,
  read it, then write a hand-tuned distroless variant.
- BuildKit is required for the build phase (Docker 18.09+ with
  `DOCKER_BUILDKIT=1`, or `docker buildx`); a Docker daemon (or an
  equivalent BuildKit endpoint) is required at build time.
- Auto-detection occasionally guesses wrong on monorepos with multiple
  apps in subdirectories — pass the explicit subdirectory as the
  positional path, and use `nixpacks.toml` to disambiguate ambiguous
  framework signals.
- The pinned `nixpkgs` revision is bumped per-nixpacks-release; pin
  the nixpacks version itself in CI for reproducible system-package
  versions across builds.
- No Windows-container support (Linux containers only).
- Custom system packages added via `--pkgs` or `nixpacks.toml [packages]`
  must be valid Nix package attribute names; the catch is that a few
  package names differ between nixpkgs and Debian (`postgresql_16`
  not `postgresql-16`, `ffmpeg-full` not `ffmpeg`). `nix search nixpkgs <name>`
  on a Nix-equipped host is the easiest lookup.

## Why it's in this catalog

The "agent generated a working app, now containerise it" step is
otherwise the longest and most error-prone step of the pipeline —
authoring a correct multi-stage Dockerfile per language is real
work that an agent gets subtly wrong (wrong base image tag, missing
system lib, wrong start command, build context too large, cache
mounts wrong). `nixpacks build .` collapses that step to one
command for the common case and is the right default for "ship the
agent's output to a registry and let CI/CD take it from there".
