# gocryptfs

> **Encrypted overlay filesystem in Go that uses FUSE to expose a
> plaintext view of an encrypted directory tree** — per-file AEAD
> (AES-256-GCM or XChaCha20-Poly1305), filename encryption with
> EME, sparse-file aware, designed specifically to back up the
> *ciphertext* directory to any cloud sync (Dropbox, NextCloud,
> rclone-mounted S3) without leaking content or filenames.
> Pinned to **v2.5.7**
> ([LICENSE](https://github.com/rfjakob/gocryptfs/blob/master/LICENSE),
> MIT).

Source: <https://github.com/rfjakob/gocryptfs>

## TL;DR

`gocryptfs` is the right answer to "I want my Dropbox / iCloud /
Google Drive folder to hold encrypted bytes the cloud provider
cannot read, but I want to use it from a regular file manager
with regular `open()` / `read()` / `write()` syscalls." It works
by mounting a FUSE filesystem at one path (the **plaintext**
mountpoint) that transparently encrypts to / decrypts from
another path (the **ciphertext** directory). Every file is
encrypted independently with AES-256-GCM (default) or
XChaCha20-Poly1305 (`-xchacha`, faster on CPUs without AES-NI),
chunked into 4 KiB blocks each with its own nonce + auth tag, so
random-access reads work without decrypting the whole file and
in-place writes only re-encrypt affected blocks (essential for
the cloud-sync use case — touching one byte of a 1 GB file
re-uploads a few KB, not the whole gigabyte). Filenames are
encrypted with EME (ECB-Mix-ECB) so they are deterministic per
directory (same plaintext name → same ciphertext name → cloud
dedup still works) but reveal nothing across directories. The
ciphertext directory is just a regular directory tree of regular
files — no block device, no loop mount, no kernel modules — so
it backs up, syncs, snapshots, and restores with any tool that
handles ordinary files. The format is a deliberate evolution of
EncFS with the security holes (chosen-ciphertext, file-size
oracle, no MAC) closed; the project ships a documented threat
model and a reproducible audit (Defuse Security 2017) is the
historical baseline.

## Install

```bash
# Homebrew (macOS — uses macFUSE under the hood)
brew install --cask macfuse
brew install gocryptfs

# Debian / Ubuntu
apt install gocryptfs

# Fedora
dnf install gocryptfs

# Arch
pacman -S gocryptfs

# Go (build from source — needs Go 1.21+ and libssl headers
# only on platforms where the OpenSSL backend is faster than
# Go's stdlib crypto)
git clone https://github.com/rfjakob/gocryptfs
cd gocryptfs && ./build-without-openssl.bash

# verify
gocryptfs --version
```

## Example usage

```bash
# init a new encrypted directory (interactive password prompt;
# writes the key-encrypted master key to cipher/.gocryptfs.conf)
mkdir -p ~/Dropbox/vault.cipher ~/vault
gocryptfs -init ~/Dropbox/vault.cipher

# mount the plaintext view at ~/vault
gocryptfs ~/Dropbox/vault.cipher ~/vault

# now ~/vault behaves like any directory; everything written
# there shows up encrypted under ~/Dropbox/vault.cipher and
# Dropbox syncs the ciphertext.
echo "secrets" > ~/vault/notes.txt
ls ~/Dropbox/vault.cipher
# gocryptfs.conf  gocryptfs.diriv  yT8K2x1oP9...   (encrypted name)

# unmount
fusermount -u ~/vault            # Linux
umount ~/vault                   # macOS

# reverse mode — present an existing plaintext directory as
# encrypted (useful for backing up an unencrypted ~/Documents
# tree to cloud storage without re-encrypting at rest)
gocryptfs -reverse ~/Documents ~/Documents.cipher
rclone sync ~/Documents.cipher remote:backup/

# change the password (re-wraps the master key, no data
# re-encryption needed)
gocryptfs -passwd ~/Dropbox/vault.cipher

# print master key for emergency recovery (store in a password
# manager; can mount with -masterkey if .gocryptfs.conf is lost)
gocryptfs -printmasterkey ~/Dropbox/vault.cipher
```

## When to choose vs alternatives

Pick **gocryptfs** over [`age`](../age/) /
[`sops`](../sops/) when the access pattern is "edit files
normally with normal tools" (age/sops are file-at-a-time
encrypt/decrypt — every save is a full round-trip; gocryptfs is
mounted and transparent). Pick over **EncFS** in every new
deployment — gocryptfs explicitly fixes the security holes the
2014 EncFS audit identified (per-block authentication, no
file-size oracle, no chosen-ciphertext attacks); EncFS is in
maintenance and should not be picked for new repositories. Pick
over **CryFS** when you want each plaintext file to map to one
ciphertext file (so cloud sync diffs work cleanly per-file) —
CryFS instead splits everything into fixed-size encrypted blocks
that hide directory structure and file sizes (better metadata
hiding, worse cloud-sync deduplication). Pick over LUKS / dm-crypt
when the requirement is per-user encrypted folders on a shared
machine, or any setup where root is not available, or where the
backing store needs to be a regular sync-able directory tree
(LUKS encrypts whole block devices and is the right answer for
full-disk encryption, not for "my Dropbox folder"). Pick over
[`rclone crypt`](../rclone/) when you want a real mounted
filesystem with `mmap()` / `O_APPEND` / sparse-file semantics —
rclone crypt is best when the data only ever lives at the remote
and never needs random-access local edits. Pairs with
[`restic`](../restic/) / [`duplicacy`](../duplicacy/) (which
deduplicate the encrypted ciphertext tree → already-encrypted
data has no compressibility, so this stack is mostly about
metadata hiding rather than dedup gains).

## Caveats

- **FUSE dependency**: needs the kernel FUSE module on Linux,
  macFUSE (kext on Intel; system extension on Apple Silicon — a
  reboot + Recovery-mode security relaxation may be required),
  WinFsp on Windows. macFUSE specifically has a non-OSI license
  for commercial use.
- **Filename length overhead**: encrypted filenames are ~1.4×
  the plaintext length (Base64), so plaintext filenames longer
  than ~176 chars hit FAT/NTFS / iCloud Drive's 255-byte limit
  on the ciphertext side.
- **Not constant-time on file sizes**: an observer of the
  ciphertext directory can see file sizes (rounded up to the
  nearest 4 KiB block + 32-byte header). If hiding file sizes
  matters more than per-file sync efficiency, pick CryFS.
- **Reverse mode is not a substitute for backup encryption with
  a HSM-stored key**: the password is on the same machine as the
  plaintext. For the "encrypt before leaving the laptop" use
  case it is the right tool; for "encrypt with a key the laptop
  cannot decrypt without an external token" use `age` with a
  YubiKey-PIV identity instead.
