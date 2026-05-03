# slim

> **Container image minifier — strip a 1.2 GB image to ~30 MB without
> rewriting the Dockerfile.**
> A single Go binary (formerly `docker-slim`) that runs your image,
> traces every syscall + every file the process actually touches via
> `ptrace` + fanotify, then synthesizes a new minimal image
> containing only those artifacts plus the original entrypoint /
> healthcheck / labels / exposed ports — typically 10–30× smaller,
> with the unused interpreters, package managers, shells, manpages,
> docs, and build leftovers gone. Pinned to **1.40.11**
> ([LICENSE](https://github.com/slimtoolkit/slim/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/slimtoolkit/slim>

## TL;DR

The "right size" for a container image is "the binary, its dynamic
libraries, the data files it reads at runtime, and nothing else."
The actual size is "Debian / Ubuntu / Alpine base layer + apt cache
+ build-time toolchain + every transitive `RUN apt-get install`
left from a Dockerfile written by someone three jobs ago." `slim`
closes that gap empirically: it boots the image, exercises it
(via `--http-probe`, a custom command, or your own integration
test), records what touched the filesystem, and emits a new
single-layer image whose contents are exactly that set. The base
image essentially disappears; the attack surface shrinks
proportionally (no `bash`, no `apt`, no `curl` for an attacker to
pivot through); pull / push / cold-start time drops in line with
the new size.

## Install

```bash
# Homebrew
brew install slim

# Linux / macOS install script (writes to /usr/local/bin)
curl -L -o ds.tar.gz \
  https://github.com/slimtoolkit/slim/releases/download/1.40.11/dist_linux.tar.gz
tar -xvf ds.tar.gz && sudo mv dist_linux/slim* /usr/local/bin/

# verify
slim --version    # slim version 1.40.11
```

Requires a working Docker / Podman daemon — `slim` drives the
runtime to spin up the original image, observe it, then `docker
commit` the squashed result.

## License

Apache-2.0 — see [LICENSE](https://github.com/slimtoolkit/slim/blob/master/LICENSE).
Permissive, patent grant included; safe to embed in commercial
build pipelines.

## One Concrete Example

```bash
# Original image: a Python web service on python:3.12 (~1.05 GB)
docker images myapp:fat
# REPOSITORY   TAG   SIZE
# myapp        fat   1.05GB

# Profile + minify in one shot. --http-probe hits common HTTP paths
# during the trace window; --include-path keeps anything you know
# the runtime touches that the probe might not exercise.
slim build \
  --target myapp:fat \
  --tag    myapp:slim \
  --http-probe true \
  --http-probe-cmd '/health' \
  --http-probe-cmd '/api/v1/ping' \
  --include-path /etc/ssl/certs \
  --include-path /usr/share/zoneinfo \
  --continue-after 30

docker images myapp:slim
# REPOSITORY   TAG    SIZE
# myapp        slim   38MB        # 27× smaller, same behaviour

# Drive the trace from your own integration tests instead of the
# built-in HTTP probe (the right answer for non-HTTP services).
slim build \
  --target          myapp:fat \
  --tag             myapp:slim \
  --exec            'pytest tests/integration -x' \
  --continue-after  signal

# Inspect what slim found inside the image (reverse-Dockerfile,
# package list, shell history, exposed ports, identified secrets).
slim xray --target myapp:fat
```

## Niche It Fills

**The "I cannot rewrite this Dockerfile" minifier.** Distroless,
Alpine, [`apko`](../apko/), Wolfi, [`melange`](../melange/),
multi-stage builds with `--from=scratch` are the right answer
*when you control the Dockerfile and can rebuild*. `slim` is the
right answer when you have an image (vendor-supplied, legacy, ML
framework with an opaque dependency tree, anything where the
build is not yours to refactor) and you need it smaller *now*.
Pairs naturally with [`dive`](../dive/) (inspect what is in a
fat image), [`syft`](../syft/) (SBOM the result), and
[`trivy`](../trivy/) / [`grype`](../grype/) (scan the slimmed
image — fewer packages = fewer CVEs by construction).

## Why use it

Three things `slim` does that hand-rolled multi-stage builds do
not, that pay back the integration cost:

1. **Empirical, not declarative.** Multi-stage builds require you
   to *know* every runtime dependency in advance — miss one and
   the container starts then dies on the first request that hits
   that code path. `slim` observes what the running image
   actually touched on a representative workload, so the
   inclusion list is derived from execution, not from reading
   `requirements.txt` and hoping. The `--include-path` /
   `--include-bin` / `--include-shell` knobs let you patch the
   inevitable "the trace did not exercise this code path" gaps
   without rerunning from scratch.
2. **Vendor / framework images become tractable.** A 4 GB
   PyTorch / CUDA / cuDNN image with a 10 MB inference script
   dropped on top is the canonical hard case — you cannot rewrite
   `nvidia/cuda:12.4.1-cudnn-runtime-ubuntu22.04` and you do not
   want to. `slim build --target your-pytorch-image:latest --exec
   'python infer.py sample.jpg'` produces a 200–400 MB image with
   the same script, same CUDA libs that script actually loaded,
   same model weights, and nothing else — with no Dockerfile
   changes.
3. **Reverse-Dockerfile + xray for free.** `slim xray --target
   image:tag` reports every layer's command, package list, file
   tree diff, exposed ports, environment, and any
   credential-shaped strings it found in the layers — useful
   independently of the minifier as the "what is actually in this
   container" tool that `docker history` only hints at.

For an LLM-CLI / agent workflow where the agent is asked to
"reduce our container size for cold-start latency", `slim build`
is the answer that does not require the agent to write a working
multi-stage Dockerfile from scratch — it asks the agent to write
a *workload script* (`--exec ...`) that exercises the real code
paths, which is a much smaller and safer ask.

## Vs Already Cataloged

- **Vs [`apko`](../apko/) + [`melange`](../melange/):** orthogonal.
  `apko` / `melange` build minimal images *forward* from declared
  apk packages (the right answer for greenfield); `slim` minifies
  *backwards* from an existing image (the right answer for legacy
  or vendor images). Use both: greenfield services on apko, vendor
  / ML / legacy images through slim.
- **Vs [`dive`](../dive/):** complementary. `dive` shows you
  *what* is bloating an image and *which layer* added it; `slim`
  removes it. The natural workflow is `dive image:fat` to
  understand, then `slim build --target image:fat` to fix.
- **Vs [`docker-slim`](https://github.com/docker-slim/docker-slim)
  (not cataloged, archived):** `slim` is the same project, renamed
  and now hosted under `slimtoolkit/`. The CLI `docker-slim` is
  still shipped for backwards compatibility but `slim` is the
  current name.
- **Vs hand-written multi-stage Dockerfiles (not cataloged):** the
  build-time path. Multi-stage is the right answer when you
  control the Dockerfile *and* you know every runtime dependency.
  `slim` is the right answer when either of those is false.

## Caveats

- **Trace coverage is the whole game.** `slim` includes only what
  it observed. Code paths your `--http-probe` / `--exec` did not
  exercise will be missing from the slim image and the container
  will fail at runtime when that path is first hit. The fix is a
  more thorough trace (run the integration suite under `--exec`,
  not just the HTTP probe) plus `--include-path` / `--include-bin`
  for known dynamic loads (templates, plugins, config dirs).
- **Dynamic loaders / plugins / late-binding break it.** Anything
  that `dlopen`s a library at first call (some Python C
  extensions, JVM dynamic class loading, Node native modules
  loaded on demand) will not be detected unless the trace
  exercises the call. Keep `--include-path` ready for these.
- **Requires a daemon.** `slim` drives Docker / Podman to run the
  source image — it is not a daemonless registry-to-registry
  tool. For pure registry plumbing of an *already-slim* image,
  reach for [`crane`](../crane/) or [`skopeo`](../skopeo/).
- **Single-layer output.** The minified image is one squashed
  layer, which makes incremental pulls (where most layers are
  cached) less efficient than a well-designed multi-layer base.
  This is the right tradeoff for cold-start / total-pull-size,
  the wrong one for "I deploy 50 micro-changes a day to the same
  base."
- **Maintenance cadence has slowed.** Last tagged release `1.40.11`
  is from 2024-02; the repo still gets commits but the release
  cadence is not what it was during the `docker-slim` era. Use
  with confidence on Docker / Podman versions current as of the
  release date; pin the binary version in CI.
