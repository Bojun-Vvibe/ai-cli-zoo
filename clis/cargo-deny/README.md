# cargo-deny

> **Cargo subcommand that lints a Rust project's dependency
> graph for license compliance, security advisories, banned
> crates, and source allowlists.** Reads `Cargo.lock`, walks
> the resolved graph, and runs four orthogonal checks
> (`advisories`, `bans`, `licenses`, `sources`) against a
> declarative `deny.toml`. Pinned to **0.19.4**
> ([LICENSE-APACHE](https://github.com/EmbarkStudios/cargo-deny/blob/main/LICENSE-APACHE),
> Apache-2.0 / MIT dual; version checked via
> `gh release view --repo EmbarkStudios/cargo-deny`).

Source: <https://github.com/EmbarkStudios/cargo-deny>

## TL;DR

`cargo deny check` is the supply-chain gate for Rust. It pulls
the RustSec advisory database, cross-references every locked
crate version, and fails the build on known CVEs, yanked
versions, or unmaintained crates. It enforces a license
allowlist (so a transitive `GPL-3.0` crate can't sneak into a
proprietary binary), bans specific crates or version ranges
(useful when a popular crate has a known footgun you want
removed organisation-wide), and pins which registries / git
sources are acceptable (no random GitHub forks pulled into
production builds). Single binary, runs in <1s on a typical
graph, designed to live in CI.

## Install

```bash
# Cargo (canonical)
cargo install --locked cargo-deny

# Pre-built binary via cargo-binstall
cargo binstall cargo-deny

# Homebrew
brew install cargo-deny

# GitHub Actions
# - uses: EmbarkStudios/cargo-deny-action@v2
#   with: { rust-version: stable }
```

## Example

```bash
# One-time scaffold of deny.toml in repo root
cargo deny init

# Run all four checks
cargo deny check

# Just the advisory check (fast in pre-commit)
cargo deny check advisories

# License audit, JSON output for downstream policy engine
cargo deny --format json check licenses

# Fail on any crate from a non-allowlisted git source
cargo deny check sources

# Workspace mode — gate every member, deny duplicates
cargo deny --workspace check bans
```

## When to use

- You ship Rust binaries or libraries to production and need a
  license SBOM gate (no surprise AGPL transitive dep).
- You want CVE alerts on your dependency graph wired into PRs,
  not a weekly Dependabot digest.
- You have an internal "no `openssl-sys`, use `rustls`" policy
  and want it enforced mechanically rather than by review.
- You vendor crates from a private registry and want to refuse
  any build that accidentally pulls from public crates.io.

## When NOT to use

- You're not in Rust — `cargo-deny` is Cargo-graph-specific.
  For Node, reach for `npm audit` / `pnpm audit` / `osv-scanner`;
  for Python, `pip-audit` / [`osv-scanner`](../osv-scanner/);
  for OS images, [`grype`](../grype/) or [`trivy`](../trivy/).
- You only need a one-off CVE check — `cargo audit` (the
  RustSec project's narrower tool) is lighter if you don't
  care about license / bans / sources.
- You want runtime sandboxing or capability enforcement —
  cargo-deny is build-time only; reach for `cap-std`,
  `wasmtime`, or seccomp profiles for runtime guarantees.

## Orthogonality vs existing zoo entries

- **vs [`osv-scanner`](../osv-scanner/)** — overlapping CVE
  surface but cargo-deny adds the license / bans / sources
  policy layer and is Cargo-native (understands features,
  workspace members, `[patch]` overrides). Use osv-scanner
  for cross-language repos, cargo-deny for Rust-only ones.
- **vs [`syft`](../syft/) + [`grype`](../grype/)** — syft
  produces an SBOM, grype scans it; cargo-deny does both for
  Cargo graphs in one pass and adds policy enforcement.
- **vs [`trufflehog`](../trufflehog/) /
  [`gitleaks`](../gitleaks/)** — those find secrets in source;
  cargo-deny finds risk in dependencies. Complementary.
- **vs [`cargo-audit`](https://crates.io/crates/cargo-audit)**
  — cargo-audit is the RustSec project's minimal advisory
  checker; cargo-deny is a superset (advisories + 3 more
  check categories) maintained by EmbarkStudios.

## Caveats

- The advisory database is volunteer-maintained; expect a small
  lag between CVE publication and a RustSec entry. Pair with
  `osv-scanner` for defence in depth.
- License detection is SPDX-string-based; some crates declare
  ambiguous combos (`MIT OR Apache-2.0 OR Zlib`) that need
  explicit allowlist clauses.
- `bans.multiple-versions = "deny"` is strict — most non-trivial
  workspaces have at least one duplicate; start with `"warn"`
  and tighten as you dedupe.
- Network egress to the advisory db / crates.io index required
  on first run; cache with `--offline` after.
