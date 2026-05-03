# cargo-binstall

- **Repo:** https://github.com/cargo-bins/cargo-binstall
- **Version:** v1.19.0 (2026-05-03)
- **License:** GPL-3.0 ([crates/bin/LICENSE](https://github.com/cargo-bins/cargo-binstall/blob/main/crates/bin/LICENSE))
- **Language:** Rust
- **Install:** `cargo install cargo-binstall` (one-time bootstrap from source) · `curl -L --proto '=https' --tlsv1.2 -sSf https://raw.githubusercontent.com/cargo-bins/cargo-binstall/main/install-from-binstall-release.sh | bash` (prebuilt) · `brew install cargo-binstall` · `winget install cargo-binstall`

## What it does

`cargo-binstall` is a `cargo` subcommand that installs Rust binaries from prebuilt release artifacts on GitHub / GitLab / Quickwit / a custom URL pattern, instead of compiling them from source — turning `cargo install ripgrep` (which would `git clone` rust-lang/rust toolchain dependencies and burn 5–15 minutes of CPU) into `cargo binstall ripgrep` (which fetches a ~3 MB tarball from the project's GitHub release and unpacks the binary into `~/.cargo/bin/` in seconds). The lookup logic walks a configurable strategy list: (1) read the crate's metadata from crates.io, (2) check the crate's `Cargo.toml` for a `[package.metadata.binstall]` block declaring the release URL pattern (`pkg-url`, `pkg-fmt`, `bin-dir` — most popular Rust CLIs ship this metadata now: `ripgrep`, `fd-find`, `bat`, `eza`, `zoxide`, `gitui`, `bottom`, `dust`, `hyperfine`, `tokei`, `cargo-watch`, `cargo-edit`, `cargo-nextest`, `sccache`, etc.), (3) substitute `{name}`, `{version}`, `{target}` (rustc target triple of your host — `aarch64-apple-darwin`, `x86_64-unknown-linux-musl`, etc.), `{archive-suffix}` into the URL pattern, (4) HEAD-check the resulting URL across each configured signature / archive-format combination (`.tar.gz`, `.tar.xz`, `.tar.zst`, `.zip`, raw binary), (5) verify a Minisign / cosign signature if the manifest declares one, (6) extract the binary, (7) write it to `~/.cargo/bin/` and update the local `~/.cargo/.crates.toml` index so `cargo install --list` and `cargo uninstall` continue to work as if the crate had been compiled. If no prebuilt artifact matches your host triple or the metadata is missing entirely, `binstall` falls back to `cargo install` from source unless you pass `--no-fallback`. Concurrency is built in (`cargo binstall ripgrep fd-find bat eza zoxide` resolves and downloads in parallel), `--locked` honors `Cargo.lock` for reproducibility, `--git <url>` installs from a git revision via the same release-artifact path, `--manifest-path` and `--targets` cover cross-arch installs (e.g. install `x86_64-unknown-linux-musl` builds onto a build server that will distribute them). `cargo binstall --self-update` upgrades binstall in place. Configuration in `~/.config/cargo-binstall/config.toml` lets you pin defaults (`default-strategies`, `quiet`, `signature-policy`).

## When to pick it / when not to

Pick `cargo-binstall` when you want the breadth of the crates.io ecosystem ("`cargo install <anything>` works") combined with the speed and reproducibility of binary distribution ("install in 3 seconds, not 5 minutes"). Concrete cases: provisioning a new dev laptop or CI image with the rust-tools collection (`cargo binstall ripgrep fd-find bat eza zoxide bottom dust hyperfine sccache tokei cargo-watch cargo-edit cargo-nextest cargo-machete tealdeer`) — what used to be a 30-minute `cargo install` chain becomes seconds, and the resulting binaries are byte-identical to what the project's CI built and signed; CI workflows that need a Rust tool (`actions-rs/install-action` is a common wrapper around binstall under the hood — `cargo binstall --no-confirm sccache` in a `setup-rust` step is faster and more cache-friendly than building from source); Dockerfile / Nix-shell snippets that want "small image, recent binaries" without a full Rust toolchain installed — `binstall` itself can be installed via the prebuilt installer in a sub-100 MB layer, then used to fetch every other tool; environments where the security model is "binaries from upstream's own GitHub Releases, signature-verified" — `binstall` honors the `[package.metadata.binstall.signing]` block which can require a Minisign / cosign signature on every artifact. Pair with [`cargo-machete`](../cargo-machete/) (find unused deps), [`cargo-nextest`](../cargo-nextest/) (faster test runner — both binstall-able), [`cargo-watch`](../cargo-watch/), [`mise`](../mise/) / [`asdf`](../asdf/) (managed Rust toolchain — binstall takes over for crate-level binaries on top), [`sccache`](../sccache/) (build cache for the source-fallback path).

Skip `cargo-binstall` when the crate you want explicitly does not publish prebuilt binaries and `cargo install` from source is the documented path — binstall will fall back, but the win is gone, and you should just use `cargo install` directly so future readers of your provisioning script know what is happening. Skip when your environment is air-gapped or behind a strict egress proxy that does not allow GitHub Releases — pre-bake the binaries into your image instead and skip the install step entirely. Skip when you are managing a *system-wide* binary inventory across many languages and `cargo-binstall` is one of five tools you would have to script around — [`mise`](../mise/) and [`asdf`](../asdf/) cover Rust *toolchains* and many ecosystem tools through plugins, and a single `mise.toml` is more portable than per-tool installers; binstall is the right pick when "the crate is on crates.io" is the constraint, not "I want one universal version manager". Skip when you are maintaining a Rust binary distribution and the question is *how* to publish prebuilt artifacts that binstall can find — the answer is the [`cargo-dist`](https://github.com/axodotdev/cargo-dist) tool, not binstall (binstall is the *consumer*; cargo-dist is the *producer*).

## Vs already cataloged

- **Vs [`cargo-update`](../cargo-update/):** complementary. `cargo install-update` upgrades all your `cargo install`-installed crates to the latest crates.io version using whatever install method was originally used. Combine: `cargo binstall` for the initial fast install, `cargo install-update -a` for periodic refresh (which itself can be configured to use binstall via `cargo install-update --binstall ...`).
- **Vs [`cargo-edit`](../cargo-edit/) / [`cargo-machete`](../cargo-machete/) / [`cargo-nextest`](../cargo-nextest/):** orthogonal. Those are crates *you install*; binstall is the *installer*. The canonical install path for each of them is `cargo binstall cargo-edit cargo-machete cargo-nextest`.
- **Vs [`mise`](../mise/) / [`asdf`](../asdf/) / [`pkgx`](../pkgx/):** different scope. Those are general-purpose multi-language version managers. binstall is `cargo`-shaped — it installs *anything on crates.io that publishes prebuilt binaries*, which is a much larger set than mise / asdf plugin authors will ever curate. Many setups use both: mise for the Rust toolchain itself, binstall for the crate-level CLIs.
- **Vs [`sccache`](../sccache/) / [`cargo-watch`](../cargo-watch/):** complementary. sccache makes the source-fallback path cheaper when binstall has to build; cargo-watch is itself a binstall-able crate.
- **Vs [`apko`](../apko/) / [`ko`](../ko/):** different domain. apko / ko build OCI images from declarative manifests; binstall installs Rust binaries on a host. They sometimes show up in the same Dockerfile (`apko` to assemble the base image, `cargo binstall` to add Rust CLIs to it).
- **Vs the official [`cargo-dist`](https://github.com/axodotdev/cargo-dist):** producer vs. consumer. cargo-dist generates the `[package.metadata.binstall]` block and the GitHub Release artifacts that binstall then fetches.

## Caveats

- **GPL-3.0, copyleft.** The CLI itself is GPL-3.0 ([crates/bin/LICENSE](https://github.com/cargo-bins/cargo-binstall/blob/main/crates/bin/LICENSE)); the underlying `binstalk` library crates are MIT/Apache-2.0 dual-licensed (re-usable under permissive terms). Practical effect: shipping the *binstall binary itself* inside a closed-source product redistribution is restricted; using it in dev / CI / on a laptop is unaffected. If you want to embed the install logic in your own tool, depend on the `binstalk` crates rather than the CLI.
- **No prebuilt artifact ⇒ falls back to source build.** This is usually what you want, but a `cargo binstall foo` that takes 8 minutes is binstall silently falling back; pass `--no-fallback` if you want a hard failure instead.
- **Signature verification is per-crate-author opt-in.** binstall *honors* a `[package.metadata.binstall.signing]` block when present, but most crates do not yet declare one. If supply-chain integrity beyond TLS + sha256 matters, vendor the binaries you care about and pin them.
- **Target triple detection assumes you want the host triple.** Cross-arch installs (`--targets aarch64-unknown-linux-musl`) work but you must explicitly pass them; defaults install for the host that ran the command.
- **`~/.cargo/.crates.toml` is the source of truth.** binstall updates it so `cargo install --list` shows binstall-installed crates uniformly with source-installed ones; manual deletion of a binary in `~/.cargo/bin/` desyncs the index — use `cargo uninstall <crate>` instead.
- **GitHub rate limits.** Anonymous binstall calls hit GitHub's REST API for release lookup; in CI on shared runners this can occasionally rate-limit. Set `GITHUB_TOKEN` (binstall reads it automatically) on CI workflows.

## Example invocations

```bash
# Bootstrap binstall itself (one-time, prebuilt — no Rust toolchain required)
curl -L --proto '=https' --tlsv1.2 -sSf \
  https://raw.githubusercontent.com/cargo-bins/cargo-binstall/main/install-from-binstall-release.sh | bash

# Install a single crate from its GitHub Release artifact
cargo binstall ripgrep

# Install many crates in parallel, no prompts (CI / provisioning)
cargo binstall --no-confirm \
  ripgrep fd-find bat eza zoxide bottom dust hyperfine \
  sccache tokei cargo-watch cargo-edit cargo-nextest cargo-machete tealdeer

# Install but require a prebuilt artifact (no source fallback)
cargo binstall --no-confirm --no-fallback gitui

# Install a specific version
cargo binstall ripgrep@14.1.1

# Install for a target other than the host (build-server distribution)
cargo binstall --targets x86_64-unknown-linux-musl --no-confirm sccache

# Self-update
cargo binstall --self-update

# Inspect what binstall would do without installing
cargo binstall --dry-run --log-level debug eza

# Configure persistent defaults
mkdir -p ~/.config/cargo-binstall
cat > ~/.config/cargo-binstall/config.toml <<'EOF'
[default]
no-confirm = true
quiet = true
EOF
```

## Why it fits the catalog

`cargo-binstall` is the "fast crates.io binary installer" entry — distinct from general-purpose version managers ([`mise`](../mise/), [`asdf`](../asdf/), [`pkgx`](../pkgx/)) and from the `cargo install` source-build path. For AI / agent workflows it is the practical answer to "the agent suggested I install `ripgrep` / `fd-find` / `bat` / `eza` / `gitui`" — the agent emits `cargo binstall <crate>` and the install completes in seconds with a signed prebuilt binary, instead of provoking a 5-minute Rust compile that derails the chat. Pair with [`cargo-update`](../cargo-update/), [`mise`](../mise/) / [`asdf`](../asdf/) for toolchain management, and the dozens of binstall-able Rust CLIs already in this catalog.
