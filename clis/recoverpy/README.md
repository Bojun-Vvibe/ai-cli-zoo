# recoverpy

> **Interactively find and recover deleted or overwritten files
> from the terminal** — a Python TUI that scans a raw block
> device for plaintext content matching a search string, lists
> every block where the string appears, and lets you preview
> each candidate and write the chosen block back out as a
> recovered file. Pinned to **v2.3.0** (commit
> `0dbf88e9a4e7b0bfd3c1e3151fb80ac3faebd706`,
> [LICENSE](https://github.com/PabloLec/recoverpy/blob/master/LICENSE),
> GPL-3.0).

Source: <https://github.com/PabloLec/recoverpy>

## TL;DR

`recoverpy` is what you reach for the moment after `rm -rf` /
`> file` / `git checkout --` removed the only copy of something
you needed. It uses `grep -a` against the raw partition device
to find blocks that still contain known fragments of the lost
file, then opens a TUI where you walk the candidate blocks,
preview each one, and save the one that matches. Unlike most
Linux recovery tools, it works on **overwritten** files too,
not just unlinked ones, as long as the bytes are still on disk.

## Install

```bash
# pipx (recommended — isolated venv, single-command upgrade)
pipx install recoverpy

# pip (user install)
pip install --user recoverpy

# from source
git clone https://github.com/PabloLec/recoverpy && cd recoverpy
pip install .

# verify
recoverpy --version    # 2.3.0
```

`recoverpy` requires **root** (it reads raw block devices) and
the system tools `grep`, `dd`, and `lsblk`. Run as
`sudo recoverpy`. macOS is **not** supported — Linux only.

## License

GPL-3.0 — see [LICENSE](https://github.com/PabloLec/recoverpy/blob/master/LICENSE).
Copyleft: derivative works that you distribute must also be
GPL-3.0. Personal recovery use is unaffected.

## One Concrete Example

```bash
# 1. classic: "I rm'd a config file that contained 'listen 8443'"
sudo recoverpy
# → TUI prompts for partition (e.g. /dev/nvme0n1p2)
# → TUI prompts for search string ("listen 8443")
# → walks blocks, preview each, save the right one to ~/recovered/

# 2. specify partition + search string up front (skip prompts)
sudo recoverpy --partition /dev/sda1 --search "BEGIN RSA PRIVATE KEY"

# 3. log file you accidentally truncated with `> file.log`
#    — bytes still live in unallocated blocks until overwritten
sudo recoverpy -p /dev/sda1 -s "ERROR connection refused"

# 4. write recovered blocks to a custom dir (default: ~/Recovered)
sudo recoverpy -p /dev/sda1 -s "func handleRequest" --save-path /tmp/recovery
```

## Niche It Fills

**Best-effort post-deletion recovery on Linux from a search
string.** The classic forensic chain is `extundelete` /
`testdisk` / `photorec` (filesystem-aware, recover by inode or
file-type signature) — powerful but unfriendly, and they don't
help with files that were *overwritten* (e.g. `> file`,
`vim :w` of an empty buffer). `recoverpy` takes the opposite
approach: forget the filesystem, search the raw bytes for
content you know, recover that. Different tool for a different
class of accident.

## Why use it

1. **Search-string-first workflow.** You usually remember a
   line from the lost file — a comment, a unique identifier, a
   key prefix. `recoverpy` makes that the *primary* lookup,
   which matches how humans actually remember files.
2. **Recovers overwritten files, not just unlinked ones.**
   `extundelete` needs the inode to still be present.
   `recoverpy` reads the raw partition, so as long as the bytes
   haven't been *overwritten in place* (rare with COW
   filesystems, common with sequential allocation), it can
   still find them.
3. **TUI preview before save.** Each candidate block is
   shown in context before you commit, so you don't end up with
   a directory of mystery 4 KiB blocks like `photorec` produces.

## Vs Already Cataloged

- **Vs [`gomi`](../gomi/) / [`rip2`](../rip2/) /
  [`trashy`](../trashy/):** these are *prevention* tools
  (trash-cans for `rm`). `recoverpy` is *cure* — what you reach
  for after the fact, when no trash-can was in place.
- **Vs [`dust`](../dust/) / [`dua`](../dua/) /
  [`ncdu`](../ncdu/):** orthogonal — disk-usage analyzers, not
  recovery tools.
- **Vs [`fclones`](../fclones/) / [`czkawka`](../czkawka/):**
  these find duplicates that *exist*; `recoverpy` finds bytes
  that the OS already considers gone.

## Caveats

- **Root + raw block device required.** Must run as `sudo` and
  point at a real partition (`/dev/sda1`, `/dev/nvme0n1p2`),
  not a mount point. Misnamed device → no results.
- **Linux only.** Reads the Linux block-device interface; macOS
  / BSD unsupported. For macOS use `testdisk` / `photorec` from
  Homebrew.
- **Stop writing to the partition immediately.** Every write
  to the partition risks overwriting the blocks you're trying
  to recover. Best practice: boot from a USB live image and
  recover from there.
- **Encrypted volumes (LUKS / FileVault) are opaque.** Raw
  blocks are ciphertext; `recoverpy` can only search the
  *unlocked* mapper device (`/dev/mapper/...`) while it is
  open.
- **COW filesystems behave differently.** btrfs, ZFS, and
  bcachefs may have already moved the blocks via copy-on-write;
  recovery rates are highly variable on those.
- **Output is raw 4 KiB blocks.** You will usually need to
  concatenate adjacent blocks by hand to reassemble a multi-KB
  file. The tool helps you find them, not stitch them.
