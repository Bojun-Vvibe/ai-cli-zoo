# minisign

## What it does
A **dead-simple file signing tool** that produces and verifies Ed25519
signatures over arbitrary files in a single ~300 KB binary with zero
configuration. One key pair (`minisign.pub` + encrypted `minisign.key`),
one signature file (`<file>.minisig`) per signed artifact, no keyrings,
no web of trust, no PKCS#11, no certificate chains, no agent daemon.
The signature format includes a trusted comment that is itself signed
(so release notes / build metadata can't be tampered with separately
from the file), plus an optional pre-hashed mode (`-H`) that signs the
BLAKE2b digest instead of the bytes — meaning verification of multi-GB
release tarballs does not require streaming the whole file twice or
holding it in memory. Compatible on the wire with `signify` (the
OpenBSD release-signing tool of the same author's design lineage), so
`.minisig` files can be verified by `signify` with the `-G` legacy flag
and vice versa for the simple-comment subset. Used in production by
Zig's release pipeline, the Tor Browser, libsodium, dnscrypt-proxy,
WireGuard's Windows installer, and many Homebrew tap maintainers.

## Why it's interesting
Different shape from GnuPG / `gpg --sign` (full OpenPGP stack: keyrings,
subkeys, web of trust, expiry, revocation, ASCII-armored multi-MB
keyrings — powerful but the configuration surface is the reason most
projects ship `sha256sums.txt` instead of signed releases), from
[`gpg-tui`](../gpg-tui/) (a UI on top of that same gpg surface, doesn't
shrink the protocol), from [`cosign`](../cosign/) /
[`sigstore`](../rekor/) (container-image and OCI-artifact signing with a
transparency log and OIDC-bound short-lived keys; great for supply-chain
provenance, overkill for "sign this tarball"), from
[`age`](../age/) / [`rage`](../rage/) (file *encryption*, not signing —
different problem), and from raw `openssl dgst -sha256 -sign` (works
but the key format is PEM, the signature is opaque binary, there's no
trusted-comment slot, and the UX is a memorable footgun). minisign is
the *I just want to sign and verify a release tarball with one Ed25519
key, in one binary, forever* shape: pick it specifically when shipping
prebuilt binaries / firmware / models / installers from a small project
that wants verifiable releases without operating a PKI. Do **not** pick
it when you need OpenPGP interop with Linux distro maintainers (use
`gpg`), when you need transparency-log-backed supply-chain provenance
(use [`cosign`](../cosign/) +
[`rekor`](../rekor/)), or when you need encryption (use
[`age`](../age/)).

## Niche category
Minimalist Ed25519 file signing — single binary, single key pair,
single `.minisig` file, signify-compatible.

## Repo
https://github.com/jedisct1/minisign

## Version pinned
`0.12` (latest tagged release, published 2025-01-15)

## License
- SPDX: `ISC`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linux)
brew install minisign

# Debian / Ubuntu
sudo apt install minisign

# Arch
sudo pacman -S minisign

# Fedora
sudo dnf install minisign

# Build from source (libsodium >= 1.0.18, cmake)
git clone --depth 1 -b 0.12 https://github.com/jedisct1/minisign
cd minisign && mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr/local ..
make && sudo make install

# Prebuilt: https://github.com/jedisct1/minisign/releases/tag/0.12
```

## Usage examples
```sh
# One-time: generate a signing key pair (prompts for a passphrase to encrypt the secret key)
minisign -G
# -> writes ~/.minisign/minisign.key  (encrypted secret, keep private)
# -> writes ~/.minisign/minisign.pub  (public key, distribute freely)

# Sign a release tarball, with a trusted comment that is itself signed
minisign -Sm myapp-1.4.2-linux-x86_64.tar.gz \
  -t "myapp 1.4.2 release, built by CI on 2026-04-30, sha256 in CHANGELOG"
# -> writes myapp-1.4.2-linux-x86_64.tar.gz.minisig

# Verify on the user's machine using the publisher's public key string
minisign -Vm myapp-1.4.2-linux-x86_64.tar.gz \
  -P RWQf6LRCGA9i53mlYecO4IzT51TGPpvWucNSCh1CBM0QTaLn73Y7GFO3

# Verify using a pubkey file shipped alongside the binary
minisign -Vm myapp-1.4.2-linux-x86_64.tar.gz -p minisign.pub

# Pre-hashed mode: sign / verify a 5 GB ISO without re-reading it twice on verify
minisign -SHm ubuntu-server-24.04.iso
minisign -Vm ubuntu-server-24.04.iso -p ubuntu.pub
```
