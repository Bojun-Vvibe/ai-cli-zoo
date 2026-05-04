# melt

## Overview

`melt` is a small command-line tool that backs up and restores SSH private keys
using a seed phrase (BIP-39 style mnemonic). Instead of copying an opaque
`id_ed25519` file around, you derive 24 human-readable words that can be written
on paper, stored in a password manager, or split across locations. Given the
same words, `melt` reconstructs the exact same key pair, so you can recover
access to servers and git remotes from scratch.

It works on existing Ed25519 keys (the only key type currently supported) and
does not require any network access or account.

## Repo URL

https://github.com/charmbracelet/melt

## Version

v0.6.2 (released 2024-08-15)

## License

MIT — upstream LICENSE file: [`LICENSE`](https://github.com/charmbracelet/melt/blob/main/LICENSE)

## Install

Homebrew:

```sh
brew install charmbracelet/tap/melt
```

Go:

```sh
go install github.com/charmbracelet/melt/cmd/melt@latest
```

Prebuilt binaries (Linux, macOS, Windows, FreeBSD) and Debian/RPM/Alpine
packages are published on the GitHub releases page.

## Quick example

Generate a seed phrase from an existing Ed25519 key:

```sh
melt ~/.ssh/id_ed25519
```

Output is 24 words. Write them down somewhere durable.

Restore the key from that seed onto a fresh machine:

```sh
melt restore --seed "word1 word2 ... word24" ~/.ssh/id_ed25519
```

The restored key has the same public key and authenticates against the same
servers as the original.

## When to choose it

- You want a paper / vault backup of an SSH key without storing the key file
  itself.
- You frequently bootstrap new machines and want a deterministic way to
  reproduce your existing identity.
- You prefer a mnemonic over keeping a binary key blob in a password manager.
- You only need Ed25519 keys and a single, focused tool.

## When NOT to choose it

- You need to back up RSA, ECDSA, or hardware-backed keys — only Ed25519 is
  supported.
- Your environment requires keys to live exclusively in a hardware token
  (YubiKey, TPM, Secure Enclave) and never be derivable from a phrase.
- You want a full secrets manager with sharing, rotation, audit logs, etc. —
  use a dedicated vault product instead.
- Your threat model assumes the seed phrase itself can be observed; a 24-word
  phrase is as sensitive as the key it represents.
