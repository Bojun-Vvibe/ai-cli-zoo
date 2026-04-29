# age

> **A small, modern, audited file-encryption tool with a single
> opinionated wire format and tiny composable CLI** — designed by
> Filippo Valsorda as the spiritual successor to `gpg` for everyday
> file encryption. Pinned to **v1.3.1**
> ([LICENSE](https://github.com/FiloSottile/age/blob/main/LICENSE),
> BSD-3-Clause).

Source: <https://github.com/FiloSottile/age>

## TL;DR

`age` (pronounced *AH-gay*, from the Japanese 上げる "to raise / lift")
is what you reach for when you need to encrypt a file, a backup, or
a secret blob and you do **not** want to think about cipher suites,
key servers, web-of-trust, or whether the recipient has imported
your subkey. One binary, one file format, two key types
(X25519 + scrypt-passphrase), no configuration file, no key ring.
Recipients are short Bech32 strings (`age1...`) you paste into a
command line; identities are equally short strings (or SSH keys you
already have). Ciphertext is `age`-format ChaCha20-Poly1305 with
streaming AEAD, so encrypting a 50 GB tarball costs constant memory
and you can pipe it directly to `s3 cp -` or `restic backup
--stdin`. The whole CLI surface is `age` and `age-keygen` — that's
it.

## Install

```bash
# Homebrew (macOS / Linux)
brew install age

# Linux package managers
apt install age            # Debian / Ubuntu (>= 22.04)
dnf install age            # Fedora
pacman -S age              # Arch
nix-env -iA nixpkgs.age    # Nix

# Go (build from source, any platform with Go >=1.21)
go install filippo.io/age/cmd/...@v1.3.1

# verify
age --version              # v1.3.1
age-keygen --version       # v1.3.1
```

## License

BSD-3-Clause — see
[LICENSE](https://github.com/FiloSottile/age/blob/main/LICENSE).
Permissive, redistributable, vendoring the binary inside another
product is fine, no copyleft on encrypted output. The Rust port
[`rage`](https://github.com/str4d/rage) is interoperable on the
wire format and dual MIT/Apache-2.0; pick whichever binary your
distro/CI ships.

## One Concrete Example

```bash
# 1. generate an X25519 keypair (private key on stdout, public on stderr)
age-keygen -o ~/.config/age/keys.txt
# Public key: age1qzr7e7p7lzzu8qmzx9ttqz6l4w7m9p2x...etc

# 2. encrypt to a recipient (their public key — paste from their README,
#    keybase, or a `~/.ssh/authorized_keys` line)
age -r age1qzr7e7p7lzzu8qmzx9ttqz6l4w7m9p2x... \
    -o secrets.tar.gz.age secrets.tar.gz

# 3. encrypt to multiple recipients at once (any one of them can decrypt)
age -r age1abc... -r age1def... -R team-recipients.txt \
    -o release-notes.md.age release-notes.md

# 4. encrypt with an SSH public key (no separate age key needed)
age -R ~/.ssh/authorized_keys -o backup.tar.age backup.tar

# 5. decrypt (uses keys.txt automatically; or pass -i)
age -d -o secrets.tar.gz secrets.tar.gz.age
age -d -i ~/.ssh/id_ed25519 -o backup.tar backup.tar.age

# 6. passphrase-only encryption (scrypt, no keypair) — stdin/stdout pipeline
tar -czf - ~/Documents | age -p > documents.tar.gz.age
age -d documents.tar.gz.age | tar -xzf -

# 7. typical backup pattern: stream ciphertext directly to object storage
restic backup --stdin --stdin-filename db.dump < <(
    pg_dumpall | age -r age1qzr7e... )
```

## Niche It Fills

**Replace `gpg --symmetric` and `gpg --encrypt` for everyday file
and stream encryption with one small binary that does not require a
key ring, a daemon, or a config file.** `gpg` is a 30-year-old
Swiss-army knife with key servers, web-of-trust, smartcard support,
S/MIME, signing modes, and a wire format from 1998; using it for
"encrypt this file to my coworker" requires understanding all of
that even though you only need 5 % of it. `age` strips the surface
to the 5 %: one cipher suite, one wire format, two key types, no
agent, no ring, no config, recipient strings short enough to paste
into a Slack message.

## Why use it

Three things `age` does that `gpg` does not:

1. **Streaming AEAD with constant memory.** `age` chunks the
   plaintext into 64 KiB segments and authenticates each one with
   ChaCha20-Poly1305, so encrypting a 1 TB disk image costs ~1 MB
   of RAM and you can pipe directly into and out of network
   transfers (`age -d s3://bucket/db.age | psql`) without
   buffering. `gpg`'s OpenPGP wire format authenticates only at
   the end, so a corrupted byte halfway through a 50 GB stream is
   only detected after you have already written all of it to disk.
2. **Recipients are paste-able strings, identities are SSH keys
   you already have.** An `age` recipient is a 62-char Bech32
   string (`age1...`) — you paste it directly into a command line
   or a CI variable, no keyserver lookup, no `--recv-keys`, no
   trust database. Identities can be your existing
   `~/.ssh/id_ed25519`, so a team that already exchanges SSH keys
   for git access can encrypt files to each other with zero new
   key-management ceremony.
3. **Wire format is small enough to audit and stable enough to
   ship.** The format is a few KB of spec
   ([age-encryption.org/v1](https://age-encryption.org/v1)),
   cryptographer-reviewed, with explicit goals of "no negotiated
   parameters" and "no in-band cipher choice." That means there is
   no "downgrade attack" surface and no `--cipher-algo AES128`
   footgun — the format pins ChaCha20-Poly1305 + X25519 + HKDF +
   scrypt and you cannot pick anything else. The cost is zero
   configurability; the benefit is that misuse-by-flag is
   impossible.

For a CI pipeline that needs to decrypt a secret env file at
runtime, `age -d -i $AGE_KEY_FILE secrets.env.age | source
/dev/stdin` is one line and one binary, with no daemon, no
keyserver, and no chance of accidentally signing instead of
encrypting.

## Vs Already Cataloged

- **Vs [`sops`](../sops/):** different layer of the stack. `sops`
  is structure-aware (it encrypts only the *values* in a YAML /
  JSON / `.env` / INI file, leaving keys and structure
  cleartext-diffable in git) and delegates the actual crypto to
  KMS / age / PGP / Vault backends. `age` is the underlying
  primitive — a generic "encrypt this byte stream" tool with no
  notion of file structure. Use `age` for opaque blobs (tarballs,
  DB dumps, binary backups); use `sops` for config files where
  you want git diffs to remain useful and where centralized KMS
  rotation matters. Common pairing: `sops` with the `age` backend.
- **Vs [`cosign`](../cosign/):** orthogonal. `cosign` signs and
  verifies container images and OCI artifacts (provenance,
  attestations, transparency log); `age` encrypts arbitrary file
  bytes for confidentiality. They compose: sign your release tar
  with `cosign sign-blob`, then encrypt the same tar with `age`
  to a recipient list before publishing to a private S3 bucket.
- **Vs `gpg` (not cataloged):** `age` deliberately covers ~5 % of
  `gpg`'s feature surface — no signing, no key servers, no
  web-of-trust, no smartcards, no S/MIME, no subkeys. If you need
  any of those, stay on `gpg`. If you only need "encrypt file to
  recipient(s), decrypt later," `age` is the right size.
- **Vs `openssl enc` (not cataloged):** `openssl enc` is a
  primitive and famously hard to use safely (no built-in
  authentication unless you manually combine with `-aead`, no
  streaming integrity, footgun-laden default key derivation).
  `age` is what `openssl enc` should be in 2025.

## Caveats

- **No signing mode.** `age` is encryption-only by design. If you
  need authenticity-without-confidentiality (a signed-but-readable
  release artifact), use `ssh-keygen -Y sign` /
  [`signify`](https://github.com/aperezdc/signify) /
  [`minisign`](https://github.com/jedisct1/minisign) /
  [`cosign`](../cosign/), and treat encryption + signing as two
  distinct steps. The author recommends `ssh-keygen -Y sign` for
  the matching "minimal SSH-key-based signing" experience.
- **No key revocation, no rotation primitive.** A recipient string
  is a public key; if it leaks or you want to remove a recipient
  from future encryptions, you re-encrypt to a new recipient list.
  There is no CRL, no expiration field, no "this key is revoked
  as of date X" mechanism. For long-lived secrets, layer `sops`
  or a KMS on top so you have a rotation point.
- **Format is final and minimal — no metadata, no comments.** The
  ciphertext file contains only the recipient stanzas and the
  authenticated payload. No filename, no MIME hint, no
  "encrypted-by" header, no expiry. If you need any of that, wrap
  the plaintext in a tarball with the metadata before encrypting.
- **Passphrase mode uses scrypt with fixed parameters.** That is
  deliberately conservative (~1 s on a 2020 laptop), but it means
  decryption is intentionally slow and you cannot tune work
  factors per-environment. Do not use passphrase mode for
  high-volume CI decryption — use keypair mode.
- **Reference implementation is Go.** That is fine for most uses,
  but if your environment forbids Go-built statically-linked
  binaries (some hardened-Linux distros), use the Rust port
  [`rage`](https://github.com/str4d/rage), which is wire-compatible
  and produces ciphertext `age` can decrypt and vice versa.
