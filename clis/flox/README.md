# flox

- **Repo:** https://github.com/flox/flox
- **Version:** v1.11.4 (latest stable, 2026-04-17)
- **License:** GPL-2.0 ([LICENSE](https://github.com/flox/flox/blob/main/LICENSE))
- **Language:** Rust (CLI) + Nix (package layer)
- **Install:** `brew install flox` · official `.pkg` for macOS / `.deb` / `.rpm` on the GitHub release page · `curl -fsSL https://downloads.flox.dev/install.sh | sh`

## What it does

`flox` is a developer-facing front-end on top of Nix that turns the
"per-project, per-shell, per-CI reproducible toolchain" problem into a
two-command workflow. Inside any project directory you run `flox init`
(creates a `.flox/` env folder with a `manifest.toml`), `flox install
ripgrep node@22 postgresql@16` (resolves and pins exact derivations
from the Nix package set), and `flox activate` (drops you into a
sub-shell where every pinned package is on `PATH`, plus any `[hook]` /
`[profile]` / `[vars]` from the manifest fires). The same env is
shareable as a *catalog* publish — `flox push` uploads the manifest +
lock to FloxHub (or a self-hosted catalog) and any teammate can `flox
pull <user>/<env>` to materialise the byte-identical toolchain on
their laptop, in CI, or inside a container layer.

The "Nix is the right answer to reproducible dev environments but
nobody on the team will write Nix" gap — flox keeps the Nix store
underneath (so you get the full ~120k-package nixpkgs catalog,
content-addressed binaries, atomic rollbacks, multi-version coexistence
on the same host) but the user-facing surface is `install` / `list` /
`uninstall` / `activate` / `edit` / `push` / `pull` with a TOML
manifest, not `flake.nix` and `mkShell`. The manifest is checked in;
the lock file (`manifest.lock`) is checked in; no `direnv`/`nix-shell`
ceremony, no Docker layer, no language-runtime sprawl from `nvm` +
`rbenv` + `pyenv` + `goenv` + a hand-rolled `tool-versions` shim.

`flox containerize` is the second-half story — the same env that
`activate` drops you into can be exported as a minimal OCI image
(`flox containerize -o myapp:latest`) so the dev shell, the CI runner,
and the deployed container are all materialised from one manifest +
lock pair.

## Pick over / pair with

- **Pick over [`mise`](../mise/) / [`asdf`](../asdf/)** when the toolchain spans
  *system libraries* (postgres, ffmpeg, imagemagick, openssl) not just
  *language runtimes*. mise/asdf shine for "pin node + python + go"; flox
  wins when you also need linked C libraries pinned to exact ABI versions
  alongside.
- **Pick over [`devbox`](../devbox/)** when GPL-2.0 + a paid-FloxHub-hosted
  catalog is acceptable and you want the more polished CLI surface; pick
  devbox when Apache-2.0 + a fully-DIY catalog model (no hosted service)
  is the requirement. Both wrap Nix; the choice is mostly about catalog
  hosting and licensing posture.
- **Pick over [`nix`](https://nixos.org) directly** when the team will not
  write Nix expressions. flox is the "Nix without the learning cliff"
  bet — you give up the full power of the language but keep 90% of the
  reproducibility wins.
- **Pair with [`direnv`](../direnv/)** — `flox activate` integrates with
  direnv via a one-line `.envrc` (`use flox`) so `cd`-ing into the
  project auto-activates the env without remembering to type the
  command.
- **Pair with [`mise`](../mise/)** along the orthogonal axis — some teams
  use mise for the language-runtime layer and flox for the system-library
  layer; the two coexist on `PATH` cleanly.

## Caveats

- License is **GPL-2.0** for the CLI itself — safe to *use* in commercial
  projects (the env it produces is not a derivative work of the CLI),
  but redistributing a modified `flox` binary inside a closed-source
  product triggers GPL obligations. The Nix store contents follow each
  package's own license.
- The hosted catalog (FloxHub) is a **commercial service** with a free
  tier; self-hosting the catalog server is supported but is a real
  operational surface, not a one-line install. Public packages from
  nixpkgs are always available without FloxHub.
- First `flox install` on a fresh laptop downloads a Nix store layer
  (~hundreds of MB to a few GB depending on the package set) — pin the
  store path or pre-warm CI runners.
- macOS support is first-class; Linux support is first-class on x86_64
  and aarch64; Windows is **WSL2 only** (no native Windows binary).
- `flox containerize` images are minimal-by-default (no shell, no libc
  beyond what the env declared) — add `bashInteractive` / `coreutils`
  to the manifest if you want to `docker exec -it` into the resulting
  container interactively.
- The Nix store on macOS lives at `/nix` (requires the volume / synthetic
  mount the official installer sets up); uninstalling flox leaves the
  Nix store behind unless you run the official Nix uninstaller too.

## Why it's in this catalog

LLM-driven coding agents and humans both struggle with the same class
of "works on my machine" failures: a different `node` minor, a
different `openssl` ABI, a different `pkg-config` path. `flox` is the
shortest path from "the README says to install five system packages"
to "the agent ran `flox activate && pytest` and got the same answer
the human got, on a fresh CI runner, two months later" — without
asking the agent or the human to learn Nix.
