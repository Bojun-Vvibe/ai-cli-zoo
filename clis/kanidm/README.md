# kanidm

> **Modern identity management platform with a single-binary
> CLI** — a Rust IDM server (`kanidmd`) plus a thick client
> CLI (`kanidm`) that together replace the FreeIPA / 389-DS /
> OpenLDAP / Keycloak stack with one binary, one config file,
> and a CLI-first administration model. Pinned to **v1.10.0**
> (released 2026-05-01,
> [LICENSE.md](https://github.com/kanidm/kanidm/blob/master/LICENSE.md),
> MPL-2.0).

Source: <https://github.com/kanidm/kanidm>

## TL;DR

`kanidm` is the answer when "I need an identity provider for
my homelab / small team / sovereign deployment" and the only
options on the table are a 12-container Keycloak stack, a
FreeIPA install that owns the host, or hand-rolling LDAP +
OAuth2 + SSH-key distribution. The server is one statically
linked Rust binary backed by a single SQLite-derived store
(`kanidmd server`) that speaks **OAuth2 / OIDC** (full RP
flow, PKCE, refresh tokens), **LDAP** read-only compatibility
for legacy clients, **SSH public key distribution**
(`AuthorizedKeysCommand` integration), **Unix accounts** via
PAM/NSS (`kanidm_unixd`), **WebAuthn / passkeys** as a
first-class credential (not a bolt-on second factor), and a
**RADIUS** subset for Wi-Fi / VPN. The `kanidm` CLI is the
canonical admin surface — every operation the web UI does is
also a `kanidm person create`, `kanidm group add-members`,
`kanidm system oauth2 create` invocation, which means
provisioning is scriptable, version-controllable, and
reviewable. The opinionated bit is the credential model:
passwords are deprecated by default (passkeys preferred), and
groups carry capability semantics rather than just naming.

## Install

```bash
# Homebrew (macOS / Linux) — client only
brew install kanidm

# Cargo (any platform with a Rust toolchain) — client only
cargo install kanidm_tools

# Docker (server + client images)
docker pull docker.io/kanidm/server:1.10.0
docker pull docker.io/kanidm/tools:1.10.0

# Pre-built binaries (Linux x86_64 / arm64)
curl -L https://github.com/kanidm/kanidm/releases/download/v1.10.0/kanidm-x86_64-linux \
  -o kanidm && chmod +x kanidm && sudo mv kanidm /usr/local/bin/
```

## When to choose kanidm

- The deployment is a homelab, small team, or sovereign /
  air-gapped environment where Keycloak's resource footprint
  and operational surface are the cost.
- Passkeys / WebAuthn need to be the primary credential
  (kanidm treats them as first-class; many alternatives still
  treat them as a 2FA bolt-on).
- The integration target is a mix of OAuth2/OIDC web apps +
  SSH hosts + Linux PAM/NSS — kanidm covers all three from
  one server without bridges.
- CLI-first / GitOps-style provisioning matters — every admin
  action is a `kanidm` subcommand suitable for Ansible /
  shell scripts / CI.
- LDAP compatibility is needed only for *reads* from legacy
  apps (kanidm exposes a read-only LDAP face; new apps should
  use OAuth2/OIDC).

## When to pick something else

- Enterprise scale with thousands of OIDC clients, custom
  authentication flows, complex federation — Keycloak's
  ecosystem and SPI plugin model are deeper.
- The team has standardised on Active Directory and needs
  full AD semantics (kanidm is not an AD replacement).
- LDAP *write* is required by a legacy app — kanidm's LDAP
  surface is read-only by design.
- Hosted IDM is acceptable — Authentik / Auth0 / Okta /
  Entra ID remove the self-hosting burden entirely.

## Caveats

- The schema is opinionated: passwords are second-class
  citizens and the recommended onboarding path is
  passkey-first. Plan the credential rollout before
  cutting over from a password-only IDM.
- The LDAP face is *read-only* — apps that bind-and-modify
  (e.g. some self-service password reset tools) cannot
  target kanidm directly.
- Pre-1.0 deployments (this catalog has tracked the project
  since the 0.x line) needed careful upgrades; v1.x is now
  stable but always read the release notes before bumping a
  minor version on a production server.
- The Unix daemon (`kanidm_unixd`) replaces SSSD for
  PAM/NSS and is the recommended path; do not run both
  simultaneously against the same realm.
