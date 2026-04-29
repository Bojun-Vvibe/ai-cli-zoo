# sops

> **Structure-aware secrets editor for YAML / JSON / ENV / INI /
> binary files** — encrypts only the *values*, leaves the keys and
> structure git-diffable, and delegates the actual cryptography to
> KMS / age / PGP / Vault / pkcs11 backends. CNCF Sandbox project,
> originally from Mozilla. Pinned to **v3.12.2**
> ([LICENSE](https://github.com/getsops/sops/blob/main/LICENSE),
> MPL-2.0).

Source: <https://github.com/getsops/sops>

## TL;DR

`sops` is what you reach for when you want secrets to live *inside*
the same git repo as the code that consumes them — without leaking
those secrets and without losing the ability to read a `git diff`.
It walks YAML / JSON / `.env` / INI files and encrypts only the
leaf values, leaving keys and tree structure cleartext. Reviewers
can see "the `database.password` field changed in this PR" without
seeing the actual passwords; CI can decrypt to plaintext on the
fly with one IAM role; key rotation rewrites only the data-key
header, not every secret. Backends include AWS KMS, GCP KMS, Azure
Key Vault, HashiCorp Vault, [`age`](../age/), PGP, and pkcs11
HSMs — multiple backends per file means any one of them can
decrypt, which is how you do break-glass access.

## Install

```bash
# Homebrew (macOS / Linux)
brew install sops

# Linux package managers
apt install sops              # Debian / Ubuntu (>= 24.04)
dnf install sops              # Fedora
pacman -S sops                # Arch
nix-env -iA nixpkgs.sops      # Nix

# Go (build from source)
go install github.com/getsops/sops/v3/cmd/sops@v3.12.2

# Direct binary
curl -L https://github.com/getsops/sops/releases/download/v3.12.2/sops-v3.12.2.darwin.arm64 \
    -o /usr/local/bin/sops && chmod +x /usr/local/bin/sops

# verify
sops --version    # sops 3.12.2 (latest)
```

## License

Mozilla Public License 2.0 — see
[LICENSE](https://github.com/getsops/sops/blob/main/LICENSE).
File-level weak copyleft: you can use `sops` in proprietary
products freely; modifications to `sops`'s own source files must
be published under MPL-2.0, but linking it as a CLI binary or
calling it as a subprocess imposes nothing on your code. This is
the standard MPL-2.0 contract — strictly less viral than LGPL or
GPL.

## One Concrete Example

```bash
# 1. configure the project's recipient policy once (.sops.yaml at repo root)
cat > .sops.yaml <<'YAML'
creation_rules:
  - path_regex: secrets/prod/.*\.yaml$
    age:        age1prodops...,age1ciread...
    kms:        arn:aws:kms:us-east-1:111111111111:key/abcd-1234
  - path_regex: secrets/dev/.*\.yaml$
    age:        age1devteam...
YAML

# 2. create or edit a secret file (opens $EDITOR with plaintext;
#    re-encrypts on save with the rules above)
sops secrets/prod/db.yaml
# inside the editor:
#   database:
#     host:     prod-db.internal
#     password: hunter2
# saves back to disk as:
#   database:
#     host:     prod-db.internal               # cleartext, diffable
#     password: ENC[AES256_GCM,data:7Ax...,iv:...,tag:...,type:str]

# 3. read a single value (no full decrypt, no temp files)
sops -d --extract '["database"]["password"]' secrets/prod/db.yaml

# 4. decrypt-to-stdout for runtime injection
export $(sops -d --output-type dotenv secrets/prod/app.env | xargs)
exec ./myapp

# 5. encrypt an existing plaintext file in place (one-shot)
sops -e -i secrets/dev/new.yaml

# 6. rotate the data key (re-encrypts the data key under all current
#    recipients; does NOT change the underlying secret values, so this
#    is safe to run on every recipient change)
sops -r secrets/prod/db.yaml

# 7. add a new recipient to all matching files
sops updatekeys secrets/prod/*.yaml
```

## Niche It Fills

**Keep secrets in git without giving up code review.** The two
common alternatives both have failure modes: (a) a separate secrets
manager (Vault, AWS Secrets Manager) means secrets and code drift
out of sync, PRs cannot describe the secret change, and rollback
requires coordinating two systems; (b) `git-crypt` / whole-file
`age` encrypts the *file*, so a PR that rotates one password
appears as "binary blob changed" with no reviewable diff. `sops`
takes the third path: encrypt at the value level, leave structure
intact, so `git diff secrets/prod/db.yaml` shows
`-password: ENC[AES256_GCM,data:7Ax...] +password: ENC[AES256_GCM,data:9Qm...]`
and a reviewer can confirm "yes, exactly one secret rotated, no
others touched, no extra fields added."

## Why use it

Three things `sops` does that whole-file encryption does not:

1. **Per-leaf encryption preserves diffs.** Each leaf string in the
   YAML / JSON tree is encrypted independently with its own AES-GCM
   nonce. Changing one secret produces one ciphertext line change
   in the git diff; adding a new secret produces one added line;
   adding a new non-secret structural key (`replicas: 3`) shows up
   as plaintext. PR reviewers can answer "what changed?" without
   ever seeing a plaintext secret. Whole-file tools (`git-crypt`,
   `age` on the file) can only show "the encrypted blob changed,
   trust me."
2. **Pluggable backends + multi-recipient = real key management.**
   A single file can be encrypted to "AWS KMS key X **and**
   GCP KMS key Y **and** age recipient Z **and** PGP key W" —
   any one of them can decrypt. That is how you build
   break-glass: prod CI uses AWS KMS, on-call humans use their
   `age` key from a YubiKey, the disaster-recovery vault holds a
   PGP key in cold storage, all on the same file. Rotating a
   recipient is `sops updatekeys`, which rewrites only the file's
   data-key header (a few hundred bytes), not the encrypted
   payload — so a 1000-secret file rotates in milliseconds.
3. **`creation_rules` make policy declarative.** `.sops.yaml` at
   the repo root maps file-path regexes to recipient sets. New
   files in `secrets/prod/` are automatically encrypted to the
   prod recipient set when created via `sops <file>`; new files
   in `secrets/dev/` get the dev set. CI can verify with `sops
   -d --verify-only` that every file is encrypted to the right
   recipients, catching "I forgot to encrypt this" and "I
   accidentally encrypted prod secrets to a dev key" in code
   review instead of after the leak.

## Vs Already Cataloged

- **Vs [`age`](../age/):** `age` is the underlying *primitive*
  for one of `sops`'s backends. Use `age` directly when you have
  an opaque blob (tar, DB dump, binary backup) and the file is
  not itself structured. Use `sops` when the file is YAML / JSON
  / `.env` / INI and you want git diffs to remain useful. The
  modern keyless-cloud-friendly pairing is `sops` + `age`
  recipients (no KMS dependency, just a public key list); the
  cloud-native pairing is `sops` + AWS KMS / GCP KMS / Azure Key
  Vault.
- **Vs [`cosign`](../cosign/):** orthogonal concern. `cosign`
  signs container images for supply-chain provenance; `sops`
  encrypts secrets for confidentiality. They compose: `cosign`
  signs the release artifact, `sops` encrypts the secrets the
  release will load at runtime, and both live in the same repo.
- **Vs `vault` (HashiCorp, not cataloged):** Vault is a
  centralized secrets *server* — runtime API, leases, dynamic
  credentials, audit log, mTLS. `sops` is a stateless file
  encryptor — no server, no daemon, secrets at rest in git.
  Many teams use both: Vault for short-lived dynamic credentials
  (DB users with TTL), `sops` for long-lived static config
  (third-party API keys, signing certs). `sops` can also use
  Vault as a backend, getting Vault's audit and rotation while
  keeping secrets in-repo for review.
- **Vs `git-crypt` (not cataloged):** both keep secrets in git.
  `git-crypt` encrypts whole files transparently via git smudge
  filters — invisible at the working-tree layer but opaque at
  the diff layer (you cannot review a secret rotation in a PR).
  `sops` is the opposite: explicit `sops <file>` invocation but
  fully reviewable diffs. For repos where every PR is reviewed,
  `sops` wins; for repos where the encryption should be
  invisible to most workflows, `git-crypt` is simpler.

## Caveats

- **Not transparent — you must remember to invoke `sops`.** Editing
  an encrypted file with plain `vim` writes plaintext back to disk
  and you will commit secrets in cleartext. Use the editor wrappers
  (`sops <file>` opens `$EDITOR` and re-encrypts on save), pre-commit
  hooks (the [sops-pre-commit](https://github.com/k8s-at-home/sops-pre-commit)
  family), and a `.gitattributes` `diff` filter that blocks plaintext
  commits. The error mode is silent and bad — design around it.
- **Key list lives in the file header, so the recipient set is
  visible.** `sops updatekeys` rotates the *data key* under new
  recipients but cannot retroactively hide who *used to* have access
  — git history records every previous recipient set. For
  high-confidentiality recipient lists, encrypt the whole file with
  `age` instead.
- **Backends are not equally robust.** AWS KMS, GCP KMS, Azure Key
  Vault, `age`, and PGP are first-class. Vault and pkcs11 work but
  have more failure modes (Vault token expiry, HSM driver quirks).
  Pin a primary backend per environment and treat alternates as
  break-glass only.
- **Binary file support is "encrypt as a single base64 blob."**
  That works but loses the per-leaf-diff property — a binary file
  in `sops` is no better than `age` on the same file. Reserve `sops`
  for structured text formats.
- **MPL-2.0 means modifications to `sops` itself must stay open.**
  Calling the binary or vendoring the library imposes nothing on
  your code, but if you fork `sops` to add a custom backend, that
  fork's source must be published under MPL-2.0. Most users only
  call `sops`, so this is rarely a concern.
