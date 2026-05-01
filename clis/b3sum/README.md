# b3sum

> **The reference command-line tool for the BLAKE3 cryptographic
> hash** — a single Rust binary that hashes files, directories,
> or stdin at multi-GB/s on a laptop CPU thanks to BLAKE3's
> SIMD-vectorised tree-hash construction, with a `sha256sum`-
> compatible output format so it drops into existing checksum
> pipelines.
> Pinned to **v1.8.5**
> ([LICENSE_A2](https://github.com/BLAKE3-team/BLAKE3/blob/master/LICENSE_A2),
> Apache-2.0; with [LICENSE_CC0](https://github.com/BLAKE3-team/BLAKE3/blob/master/LICENSE_CC0)
> and [LICENSE_A2LLVM](https://github.com/BLAKE3-team/BLAKE3/blob/master/LICENSE_A2LLVM)
> for the optional alternative grants).

Source: <https://github.com/BLAKE3-team/BLAKE3>

## TL;DR

`b3sum` is `sha256sum` for the era when SHA-256 has become the
slowest single-core step in your build/CI/backup pipeline. Same
on-the-wire UX (`b3sum file > file.b3`, `b3sum -c file.b3`),
same exit codes, same `--check` flag — but ~10-20× faster on a
modern x86_64 / aarch64 chip because BLAKE3 hashes 1024-byte
chunks in parallel using AVX-512 / AVX2 / SSE4.1 / NEON, then
combines them into a Merkle tree. The output is 256 bits by
default but the function is XOF (extendable-output), so
`b3sum --length 64` gives you 64 bytes of derived key material
in one call without HKDF.

## Install

```bash
# Homebrew (macOS / Linux)
brew install b3sum

# Cargo (any platform with Rust 1.66+)
cargo install b3sum

# Linux package managers
# Arch:           pacman -S b3sum
# Debian/Ubuntu:  apt install b3sum
# Fedora:         dnf install b3sum
# Alpine:         apk add b3sum
# Nix:            nix-env -iA nixpkgs.b3sum

# pre-built binary release (Linux x86_64 / aarch64, macOS arm64,
# Windows x86_64)
curl -LO "https://github.com/BLAKE3-team/BLAKE3/releases/download/1.8.5/b3sum_x86_64-unknown-linux-musl"
chmod +x b3sum_x86_64-unknown-linux-musl
sudo mv b3sum_x86_64-unknown-linux-musl /usr/local/bin/b3sum

# verify
b3sum --version    # b3sum 1.8.5
```

One static binary, ~3 MB. No dependencies, no daemon, no
config.

## License

Apache-2.0 (default) — see
[LICENSE_A2](https://github.com/BLAKE3-team/BLAKE3/blob/master/LICENSE_A2).
The repo also offers the source under CC0
([LICENSE_CC0](https://github.com/BLAKE3-team/BLAKE3/blob/master/LICENSE_CC0))
and a separate Apache-2.0 grant for the LLVM-derived bits
([LICENSE_A2LLVM](https://github.com/BLAKE3-team/BLAKE3/blob/master/LICENSE_A2LLVM));
downstream picks whichever is convenient. All three are
permissive — embedding the binary or the C / Rust libraries is
fine for proprietary use.

## One Concrete Example

```bash
# 1. drop-in for sha256sum
b3sum *.tar.gz > SHA.b3
b3sum -c SHA.b3            # verify all listed files

# 2. hash a directory tree (b3sum walks paths from argv only;
#    use find for recursion, like sha256sum)
find dist -type f -print0 | xargs -0 b3sum > dist.b3
b3sum -c dist.b3

# 3. hash from stdin (the dest pattern for streaming)
tar c src/ | b3sum
xz -dc backup.tar.xz | b3sum    # verify a downloaded archive

# 4. parallel multi-file hashing (b3sum reads files in parallel
#    by default; for explicit control, use xargs -P)
find /var/log -type f | xargs -P 8 -n 1 b3sum > logs.b3

# 5. extendable output (XOF) — derive 64 bytes of key material
b3sum --length 64 --no-names < secret.bin

# 6. keyed BLAKE3 (MAC) — keyed hash, like HMAC but native
b3sum --keyed --no-names < message.bin   # reads 32-byte key from
                                         # the BLAKE3_KEY env file

# 7. KDF mode — derive a subkey from a master key + context
b3sum --derive-key "myapp 2026 session keys v1" \
      --length 32 --no-names < master.key

# 8. raw bytes instead of hex (good for piping into another tool)
b3sum --raw --no-names < input.bin | hexdump -C

# 9. no-mmap mode (forces streaming reads; useful for special
#    files like /dev/sda or NFS that misbehave under mmap)
b3sum --no-mmap < /dev/sda1
```

## Niche It Fills

**A modern hash function with a UX identical to `sha256sum`.**
Most of the catalog already assumes SHA-256 in checksum lines
(release manifests, lockfiles, reproducible-build attestations).
The CPU cost is real: a 4 GB tarball hashed with `sha256sum`
takes ~12 s on one core of a Ryzen 7700; `b3sum` does the same
in ~1 s using AVX-512 plus internal multi-threading. For build
caches, content-addressed object stores, file-deduplication
backups, and "did this 30 GB Docker image change" pipelines,
that delta dominates wall-clock time. b3sum is the smallest
possible swap-in to capture it without redesigning the pipeline.

## Why use it

Three things `b3sum` does that picking another fast hash does
not, that explain why it earned a separate entry alongside
[`sha256sum`/`shasum`-style classics not cataloged]:

1. **One function for hash + MAC + KDF + XOF.** SHA-256 needs
   HMAC-SHA-256 for MACs and HKDF for KDFs and `SHAKE128`/
   `XOF` for extendable output — three primitives, three
   command paths, three opportunities to misuse. BLAKE3 has all
   four modes built into the same construction, exposed as
   `--keyed`, `--derive-key`, `--length`, `--raw` flags. One
   tool, one binary, one set of test vectors.
2. **Multi-threaded by default, on a single file.** Other fast
   hashes (xxHash, CityHash) are fast per-thread but linear; to
   parallelise you split the file into chunks yourself and
   combine. BLAKE3's tree structure lets `b3sum` use all cores
   on a single-file hash automatically — `b3sum 30G.bin` lights
   up 16 threads. Compare `sha256sum` pegging one core at 100%.
3. **Cryptographically reviewed, not just fast.** xxHash and
   CityHash are *not* cryptographic — they are not collision-
   resistant against an adversary, and using them for content-
   addressed storage where attackers can choose inputs is
   unsafe. BLAKE3 is built on the BLAKE2 lineage (NIST SHA-3
   finalist) by Jean-Philippe Aumasson, Samuel Neves, Zooko
   Wilcox-O'Hearn, and Jack O'Connor, with a public audit
   record. You get the speed of a non-crypto hash with the
   guarantees of a crypto one.

For an LLM-CLI workflow, `b3sum` is the right primitive for
content-addressing model outputs, prompt files, retrieved
documents, and tool-call results in a deterministic-cache layer
— the per-call overhead is negligible even on tiny inputs.

## Vs Already Cataloged

- **Vs [`hexyl`](../hexyl/):** different niche entirely — hexyl
  is a hex viewer (binary content inspector); b3sum is a hash.
  Pair them: `b3sum file && hexyl file | head` is the standard
  "what is this file and what's its identity" combo.
- **Vs `sha256sum` / `shasum -a 256` (not cataloged because they
  ship with every Unix):** Same UX, same `-c` verification
  semantics, ~10-20× faster on multi-MB inputs and with the
  bonus features above. The migration is literally `s/sha256sum/
  b3sum/` in scripts; the only change is the algorithm name in
  manifests (and a `--length 32` if you want to keep 256-bit
  output, which is the default).
- **Vs `openssl dgst -sha3-256` (not cataloged):** SHA-3 is a
  Keccak sponge, also faster than SHA-2 on some platforms but
  much slower than BLAKE3, and the `openssl` UX is awkward for
  the `-c` verification flow. Pick SHA-3 only if a standard
  forces it; otherwise BLAKE3 wins on speed and ergonomics.
- **Vs xxHash (xxh3, not cataloged):** xxHash is even faster
  on a single thread (~30+ GB/s) but is not a cryptographic
  hash. Pick xxHash for in-process dedup keys and content
  fingerprints inside trusted boundaries (build cache keys,
  hashmap shards); pick BLAKE3 wherever an attacker could plant
  inputs (release manifests, content-addressed storage,
  signature inputs).
- **Vs Git's SHA-1 → SHA-256 transition (not cataloged):** Git
  is moving toward SHA-256 for object names; BLAKE3 has been
  proposed but is not the chosen successor. Outside Git, BLAKE3
  is uncontroversially the right pick for new content-addressed
  systems.

## Caveats

- **Output is not a `sha256sum` checksum.** A `b3sum` line
  starts with a different hex digest (256 bits = 64 hex chars,
  same as SHA-256, but the value is different). Mixing
  `b3sum -c` with a SHA-256 manifest fails the check. Pick one
  algorithm per manifest.
- **Default 256-bit output is by convention only.** BLAKE3 is
  XOF, so somebody at the other end could write `b3sum --length
  16 file.b3` and produce a 128-bit hash that still verifies
  with `b3sum -c` on a matching manifest. If you need a fixed
  output length, document it (`# b3sum 256-bit` in the file
  header) and validate.
- **No directory recursion verb.** Like `sha256sum`, `b3sum`
  hashes only the paths you pass. Use `find … -print0 | xargs
  -0 b3sum` for recursive walks. (Or use `b3sum --num-threads N`
  to control the per-file parallelism inside that loop.)
- **mmap mode can misbehave on special files.** `b3sum`
  defaults to memory-mapping inputs for speed; `/dev/sda1`,
  some FUSE filesystems, and some NFS mounts return wrong sizes
  under mmap. Use `--no-mmap` if a hash unexpectedly differs
  from a streaming tool.
- **CPU feature detection is runtime, not build-time.** A
  binary compiled on a Skylake host runs on a Zen 3 host fine,
  but if you static-link your own program against `blake3` and
  forget to enable runtime dispatch, you'll get the portable
  reference impl (still correct, ~5× slower than vectorised).
  Check `b3sum --version` if performance is suspicious — it
  reports the chosen SIMD path.
- **Keyed mode requires exactly 32 bytes of key.** Shorter or
  longer key files fail with a clear error; do not pre-hash a
  passphrase to "derive" a key. For passphrase-derived keys,
  use `--derive-key` mode (which is the documented KDF
  primitive) instead of `--keyed`.
