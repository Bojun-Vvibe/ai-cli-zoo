# rage

> **A Rust implementation of `age` — a simple, secure, modern
> file encryption tool with small explicit keys, no config knobs,
> and UNIX-style composability — distributed as two single-binary
> CLIs (`rage` for encrypt/decrypt, `rage-keygen` for key
> generation).** Pinned to **v0.11.2**, dual-licensed
> Apache-2.0 OR MIT
> ([LICENSE-APACHE](https://github.com/str4d/rage/blob/main/LICENSE-APACHE),
> [LICENSE-MIT](https://github.com/str4d/rage/blob/main/LICENSE-MIT)).

- **Repo:** https://github.com/str4d/rage
- **Latest version:** v0.11.2 (2026-04-22)
- **License:** Apache-2.0 OR MIT (dual; pick either at use; SPDX
  `Apache-2.0` and `MIT`; files at repo root)
- **Category:** `crypto` / `encryption` / `secrets`
- **Language:** Rust

## What it does

`rage-keygen -o key.txt` writes an X25519 keypair (public key
echoed to stderr, private key to the file).
`rage -r age1abcd...xyz -o secret.age secret.txt` encrypts to a
recipient public key.
`rage -d -i key.txt secret.age > secret.txt` decrypts using the
matching identity. Recipients can also be passphrases (`-p`),
SSH keys (`ssh-rsa` / `ssh-ed25519` lines from
`~/.ssh/authorized_keys`), or YubiKeys via the `age-plugin-yubikey`
plugin (rage transparently shells out to `age-plugin-*` binaries
on PATH per the `age` plugin spec). Multiple recipients are
additive — `rage -r alice -r bob -r carol -o blob.age data.bin`
produces one ciphertext that any of the three can decrypt
independently. The format is **streaming** (you can pipe
`tar c . | rage -r ... > backup.tar.age` and decrypt
back through `tar x` without buffering the whole archive),
authenticated (ChaCha20-Poly1305), and stable since age v1
(spec: <https://age-encryption.org/v1>) — `rage` and Go `age`
ciphertexts are wire-compatible in both directions.

## Why included

Already cataloged: [`age`](../age/) (the upstream Go
implementation by FiloSottile + Ben Cartwright-Cox).
`rage` is included as the **Rust-native parallel** for two
specific cases: (1) Rust-only build pipelines / containers that
want one less Go-toolchain dependency in the image, and (2)
environments that need the YubiKey / TPM / FIDO2 plugins that
landed first or work better in the Rust ecosystem
(`age-plugin-yubikey` is itself Rust). Same wire format, same
keys, same `age1...` recipient strings — `age` and `rage`
interoperate byte-for-byte, so picking one is a build-system
concern, not a security or compatibility one. For an LLM-CLI
workflow that ships encrypted artifacts between agent steps
(plan output, tool credentials, model snapshots) on a
Rust-toolchain build host, `cargo install rage` is the
single-command path to the same `age1...`-keyed secret store
the rest of the team already uses with Go `age`.
