# pdm

> **A modern Python package + dependency manager that speaks
> PEP 517 / PEP 518 / PEP 621 / PEP 582 natively — `pyproject.toml`
> is the single source of truth, `pdm.lock` is a cross-platform
> resolved lockfile, and the resolver runs on a custom
> SAT-style backtracker that beats `pip`'s naive walker on
> conflict-heavy graphs.** Pinned to **v2.26.8**,
> [LICENSE](https://github.com/pdm-project/pdm/blob/main/LICENSE),
> MIT.

Source: <https://github.com/pdm-project/pdm>

## TL;DR

`pdm` is the answer when you want `poetry`'s "one tool, one
config file, one lockfile" workflow but without `poetry`'s history
of resolver pain and without inventing a private metadata format.
`pyproject.toml` follows PEP 621 verbatim — `[project]`,
`[project.optional-dependencies]`, `[project.scripts]` — so
anything that reads PEP 621 (uv, hatch, build, twine, ruff) reads
your `pdm` project unchanged. The lockfile (`pdm.lock`) is
cross-platform: one resolution result that works on Linux, macOS,
and Windows, with markers carrying the per-platform branches.
`pdm install` is fast (parallel downloads, wheel cache shared
across projects via a content-addressed store), and `pdm add foo`
does what `pip install foo && pip freeze > requirements.txt`
*should* have done — resolves, locks, installs, updates
`pyproject.toml`, in one command, deterministically. The novelty
shipped in 2020 was PEP 582 (`__pypackages__/` instead of a
virtualenv); the long-term win has been a clean, plugin-friendly
core that scripts well.

## Install

```bash
# pipx (recommended — isolated install)
pipx install pdm

# Homebrew (macOS / Linux)
brew install pdm

# install script (Linux / macOS, official)
curl -sSL https://pdm-project.org/install-pdm.py | python3 -

# Pre-built standalone binary
curl -L "https://github.com/pdm-project/pdm/releases/download/2.26.8/pdm-2.26.8-$(uname -m)-apple-darwin.tar.gz" \
  | tar xz && sudo mv pdm /usr/local/bin/

# Conda
conda install -c conda-forge pdm

# verify
pdm --version    # PDM, version 2.26.8
```

## License

MIT — see
[LICENSE](https://github.com/pdm-project/pdm/blob/main/LICENSE).
Permissive, no copyleft, redistribute the binary and embed in
internal toolchains freely.

## One Concrete Example

```bash
# 1. start a fresh project (interactive, picks Python, picks build backend)
mkdir myproj && cd myproj
pdm init

# 2. add runtime + dev dependencies — locks deterministically, installs in parallel
pdm add 'httpx>=0.27' 'pydantic>=2'
pdm add -dG test pytest pytest-cov ruff
pdm add -dG docs mkdocs-material

# 3. run scripts and tools inside the managed env (no `source venv/bin/activate`)
pdm run pytest -q
pdm run ruff check .

# 4. inspect / update / sync
pdm list --tree                  # full dep tree, version + license per node
pdm outdated                     # which deps have newer compatible releases
pdm update --update-eager        # respect ranges, pull latest within them
pdm sync --clean                 # drop anything not in pdm.lock (CI-style)

# 5. CI gate: install from lockfile only, fail on any mismatch
pdm install --frozen-lockfile --no-self --no-editable

# 6. publish a package (uses pyproject.toml [project] + chosen build backend)
pdm build
pdm publish --repository pypi

# 7. lock against a frozen registry snapshot for reproducibility
pdm lock --exclude-newer 7d      # pretend the index is 7 days old
```

## Niche It Fills

**A standards-tracking Python project manager that reads and
writes the same `pyproject.toml` everyone else does, with a
cross-platform lockfile and a fast resolver, MIT-licensed.** The
crowded space — `pip` + `pip-tools`, `poetry`, `hatch`, `pipenv`,
`rye`, `uv`, `conda` — has each tool stuck on one tradeoff.
`pdm`'s tradeoff is *adherence to PEPs over invention*: no
custom `[tool.poetry]` block, no parallel "Pipfile" universe, no
opinionated build backend lock-in. The resolver is fast enough
to compete with `uv` on most graphs, and the plugin API
(`pdm-plugin-hello`-style) is the most extensible of the
`pyproject.toml`-native managers.

## Why use it

Three things `pdm` does that `poetry` / `pip-tools` / `pipenv` do not:

1. **PEP 621 metadata, no proprietary block.** Your
   `pyproject.toml` `[project]` table is the ground truth —
   the same metadata `build`, `twine`, `uv`, and `hatch`
   consume. Move off `pdm` tomorrow and your manifest still
   works; `poetry` users have to translate `[tool.poetry]` →
   `[project]` first.
2. **Cross-platform lockfile with markers.** `pdm.lock` carries
   one resolution graph with per-platform branches encoded as
   PEP 508 markers, so the same lockfile works on Linux,
   macOS, Windows, and CPython / PyPy without re-resolving per
   OS. `pip-tools` requires a lockfile per platform; `poetry`
   gets close but ships its own format.
3. **First-class scripts + tasks without a Makefile.**
   `[tool.pdm.scripts]` defines named tasks (`pdm run lint`,
   `pdm run release`) with composition (`{ composite = ["lint",
   "test"] }`), env injection, and pre/post hooks. Removes one
   reason to drag `task` / `just` into a pure-Python project.

For an LLM-CLI workflow, `pdm` is the right tool to manage the
Python side of a mixed repo where the agent needs to *add a
dependency, run the tests, and commit a deterministic lock
update* in one round-trip — `pdm add foo && pdm run pytest`
leaves both `pyproject.toml` and `pdm.lock` in a state your
review tooling can diff cleanly.

## Vs Already Cataloged

- **Vs [`uv`](../uv/):** the close competitor. `uv` is the
  Astral-built Rust resolver/installer that has eaten most of
  `pip` and `pip-tools`'s mindshare in 2024-2025; it is faster
  on cold caches and has a smaller surface (no script runner,
  no plugin API yet). Pick `uv` for "fastest install, drop-in
  for `pip`," pick `pdm` for "one tool that owns the project
  lifecycle (init / add / run / build / publish) with a plugin
  ecosystem." Many shops use `uv` as the installer behind
  `pdm`'s frontend (`pdm config use_uv true`).
- **Vs [`rye`](../rye/):** also project-lifecycle-shaped. `rye`
  bundles a Python toolchain manager (downloads CPython for
  you) and is opinionated about "one way to do it." `pdm` is
  more configurable and does not ship its own Python builds —
  point it at any interpreter (`pdm use python3.12`) and go.
  `rye` is now a thin shim over `uv` upstream; `pdm` remains
  independently maintained.
- **Vs [`pipx`](https://pypa.github.io/pipx/):** orthogonal.
  `pipx` installs *applications* (one venv per CLI tool, on
  PATH); `pdm` manages *project* dependencies. Use `pipx` to
  install `pdm` itself.

## Caveats

- **Resolver can still be slow on huge graphs vs `uv`.** For a
  100+ package monorepo, `uv lock` is 2-5x faster than `pdm
  lock` on a cold cache. `pdm` ships a `use_uv` config flag
  that delegates resolution to `uv` and closes the gap, but
  you then carry two tools.
- **Lockfile format has churned across major versions.** `pdm
  2.x` lockfiles are not 1.x compatible; downgrading `pdm` in
  CI without re-locking will error. Pin the `pdm` version in
  your CI image alongside Python.
- **PEP 582 (`__pypackages__/`) is dead.** `pdm`'s original
  flagship feature — virtualenv-free local installs — was
  rejected by the broader Python community and is now a
  deprecated mode. Default install creates a normal `.venv/`;
  do not adopt PEP 582 for new projects.
- **Plugin ecosystem is smaller than `poetry`'s.** A few
  hundred `pdm-*` plugins exist (versioning, multirepo, conda
  bridge), but `poetry` still has more third-party integrations
  (Renovate templates, Dependabot ecosystem entries).
  Mainstream stuff works; long tail may not.
- **Build backend choice is yours, for better and worse.** `pdm
  init` asks which backend (pdm-backend, hatchling, setuptools,
  flit-core); the answer matters for sdist layout and metadata.
  Pick `pdm-backend` for a self-consistent stack, `hatchling`
  if you want maximum portability.
