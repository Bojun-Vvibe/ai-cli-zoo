# horcrux

> **Split a single file into N encrypted "horcrux" fragments
> any K of which reconstruct it** — Shamir-secret-shared
> file splitter that takes one input (any binary or text),
> a `--total` and `--threshold`, and produces N self-describing
> `.horcrux` shards each carrying its own header + AES-256
> ciphertext + an XOR key share; pick any K of the N shards
> back, run `horcrux bind`, and the original file falls out
> bit-identical, with no shard alone leaking anything about
> the plaintext. Pinned to **v0.2** (SPDX: `MIT`,
> [LICENSE](https://github.com/jesseduffield/horcrux/blob/master/LICENSE)).

Source: <https://github.com/jesseduffield/horcrux>

## TL;DR

`horcrux` is a single-binary Go tool that does
threshold-secret-sharing over a *file*: `horcrux -t 3 -n 5
secret.zip` splits `secret.zip` into 5 shards
(`secret_1_of_5.horcrux` … `secret_5_of_5.horcrux`), any 3
of which reconstruct the original via `horcrux bind`. Each
shard contains a JSON header (index, total, threshold,
canonical filename, OAEP nonce) followed by AES-256-OFB
ciphertext of the same plaintext, and the AES key itself is
split across the shards using Shamir's Secret Sharing — so
loss of (N−K) shards is fine and capture of fewer than K
shards reveals nothing. Built by the `lazygit` author as a
practical "split a wallet seed / a recovery key / a sensitive
archive across friends, drives, and clouds" CLI.

## Install

```bash
# Go install
go install github.com/jesseduffield/horcrux@latest

# Pre-built binary (macOS / Linux / Windows)
# https://github.com/jesseduffield/horcrux/releases/tag/v0.2

# verify
horcrux --help
```

## License

MIT — see
[LICENSE](https://github.com/jesseduffield/horcrux/blob/master/LICENSE).

## Representative Commands

```bash
# 1. split secret.zip into 5 shards, any 3 reconstruct it
horcrux -t 3 -n 5 secret.zip

# 2. distribute shards: 1 to a USB drive, 2 to clouds, 2 to friends
mv secret_1_of_5.horcrux /Volumes/usb/
rclone copy secret_2_of_5.horcrux remote-a:backup/
rclone copy secret_3_of_5.horcrux remote-b:backup/

# 3. recombine: copy any 3 shards into one directory and bind
mkdir restore && cp secret_{1,2,4}_of_5.horcrux restore/
horcrux bind ./restore   # writes secret.zip in cwd
```

## Why It Matters

Encrypting a sensitive blob with one key just moves the
problem — now you have to protect the key. Sharing the key
"in N places" trivially is also bad (any one place leaks
everything). Shamir's Secret Sharing solves this in theory,
but most implementations operate on small inputs (a 32-byte
key, a 12-word seed phrase) and leave the file-level wrapping
to the user. `horcrux` does the whole loop end-to-end on a
*file of arbitrary size*: AES-256-OFB encrypts the plaintext,
the AES key is Shamir-split into N shares, each shard
self-describes with a JSON header so reassembly works without
side-channel metadata, and the binary is small enough to
ship onto every drive and laptop holding a shard. Pick over
hand-rolling `gpg --symmetric` + `ssss` when you want one
self-contained tool that produces interchangeable shards
suitable for "store on USB sticks scattered across friends /
safety-deposit boxes / cloud accounts and recover from any
K of N if a house burns down or a cloud account dies"
disaster-recovery scenarios. Pairs with `age` / `rage` (when
the threshold scheme is overkill and a single recipient key
is enough) and any backup tool (`restic`, `borg`) that ships
encrypted archives you then want to gate behind a recovery
threshold. The killer property is **lossy-storage tolerance
without a single point of compromise**: lose `N−K` shards
and you still recover; capture `K−1` shards and the attacker
has nothing.
