# pixi

- **Repo:** https://github.com/prefix-dev/pixi
- **Version:** v0.67.2 (released 2026-04-23)
- **License:** BSD-3-Clause ([LICENSE](https://github.com/prefix-dev/pixi/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install pixi` · `curl -fsSL https://pixi.sh/install.sh | bash` · `winget install prefix-dev.pixi` · prebuilt binaries on the GitHub release page · binary name is `pixi`

## Overview

`pixi` is a Rust-implemented package manager that reuses the
**conda-forge** ecosystem (and PyPI, in the same lockfile) to
deliver fully reproducible, cross-platform project
environments without anaconda or `conda`/`mamba` themselves.
The contract is the one Cargo and pnpm have made familiar:
`pixi.toml` declares dependencies, tasks, and platforms;
`pixi.lock` pins exact resolved versions for every declared
platform (`linux-64`, `osx-arm64`, `osx-64`, `win-64`); `pixi
install` materialises the env into `.pixi/envs/default/`;
`pixi run <task>` activates the env and runs the command.

Because it draws from conda-forge, `pixi` resolves *system*
dependencies (CUDA, BLAS, ffmpeg, GDAL, PROJ, R, OpenJDK,
PostgreSQL clients) the way pip cannot — those land as
binary packages alongside Python wheels in the same lock.
`[pypi-dependencies]` then layers PyPI-only packages on top,
so a project that needs `pytorch` (conda-forge, with the
right CUDA build) and `some-niche-pypi-only-lib` declares
both in one file and locks both deterministically.

Multi-environment support means one `pixi.toml` can declare
`[feature.dev]`, `[feature.docs]`, `[feature.cuda]` etc. and
compose them into named envs (`pixi run --environment dev
test`). Tasks are first-class (`[tasks] test = "pytest -q"`)
with `depends-on` for ordered DAGs and `inputs` /  `outputs`
for skip-if-up-to-date file fingerprints à la `make`.

## Use cases

- **Reproducible Python + system-library projects.** ML
  workloads needing CUDA, geospatial work needing GDAL/PROJ,
  bio/chem pipelines needing R alongside Python — anything
  pip alone cannot resolve cleanly.
- **Cross-platform team development.** `pixi.lock` pins
  every declared platform, so a Linux CI build, a Mac dev's
  `arm64`, and a Windows contributor all materialise the
  identical resolved set without per-OS lockfile drift.
- **CI cache hits.** The lockfile + `~/.cache/pixi/` content
  store is friendly to GitHub Actions / GitLab CI cache
  keys; cold installs are fast because conda-forge mirrors
  ship prebuilt binaries (no compilation step the way
  source-pip-install would need).
- **Drop-in replacement for `conda env create -f
  environment.yml`** in projects that want a faster, lockfile-
  backed solver and a single binary instead of Anaconda's
  full distribution.

## Why pick this

Pick `pixi` when the project genuinely needs conda-forge's
binary system-library coverage (CUDA / GDAL / R / Java /
ffmpeg) and you want a Cargo-shaped UX with deterministic
locking. It is the right answer for ML/data/scientific Python
where pure-pip workflows ([`uv`](../uv/), `poetry`) hit
"install pip can't satisfy this OS dep" walls. For
pure-Python projects with no system-library dependencies,
[`uv`](../uv/) is faster and smaller; for non-Python work
nothing about `pixi` is Python-only (it can install Node, Go,
Rust, etc. from conda-forge), but the ecosystem fit is
strongest where conda-forge already has rich coverage.

## Comparable alternatives

- [`uv`](../uv/) — astral-sh's Rust pip/Poetry replacement;
  pure-Python wheels only, no system libraries; faster for
  that scope.
- `conda` / `mamba` — original conda-forge clients; `pixi`
  replaces them with a project-local lockfile model and a
  single Rust binary.
- `poetry` / `pdm` / `hatch` / `rye` — Python-only project
  managers; pick when no conda-forge system dep is needed.
- `nix` / `devbox` / `flox` — far broader reproducibility
  story (full system packages, not just language envs); pick
  when you need OS-level reproducibility, not just env-level.
- `docker` + `requirements.txt` — heavier; pick when you
  also need OS isolation, not just dep pinning.
