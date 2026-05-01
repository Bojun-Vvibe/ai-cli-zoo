# borgbackup

> **A deduplicating, encrypted, compressed backup tool that
> stores snapshots in a content-addressed repository on local
> disk, SSH target, or BorgBase/rsync.net** — one `borg create`
> writes a new snapshot in seconds because chunks already in the
> repo are not re-uploaded, and one `borg mount` exposes any past
> snapshot as a FUSE filesystem you can `cp` files out of.
> Pinned to **v1.4.4**
> ([LICENSE](https://github.com/borgbackup/borg/blob/master/LICENSE),
> BSD-3-Clause).

Source: <https://github.com/borgbackup/borg>

## TL;DR

`borg` is what you reach for when `rsync -a` to an external disk
stops being enough — when you want the last 90 daily snapshots
of `~/code` to fit in 30 GB instead of 2 TB, when you want every
byte at rest on the backup target to be encrypted, and when you
want pruning ("keep daily for 14d, weekly for 8w, monthly for
12m") to be one declarative command instead of a cron of
`find -mtime`. The repository is content-addressed: file content
is split into variable-size chunks via a rolling hash
(buzhash), each chunk stored exactly once keyed by its HMAC,
and a snapshot is a manifest of chunk IDs. So a 100 GB home
directory backed up nightly for a year typically costs 120-200
GB of repo space, not 36 TB.

## Install

```bash
# Homebrew (macOS / Linux)
brew install borgbackup

# Linux package managers
# Arch:           pacman -S borg
# Debian/Ubuntu:  apt install borgbackup
# Fedora:         dnf install borgbackup
# Alpine:         apk add borgbackup
# openSUSE:       zypper install borgbackup
# Nix:            nix-env -iA nixpkgs.borgbackup

# FreeBSD
pkg install py311-borgbackup

# pipx (any platform with Python 3.9+)
pipx install borgbackup==1.4.4

# single-file binary release (statically-linked, no Python needed)
curl -LO "https://github.com/borgbackup/borg/releases/download/1.4.4/borg-linux-glibc236"
chmod +x borg-linux-glibc236
sudo mv borg-linux-glibc236 /usr/local/bin/borg

# verify
borg --version    # borg 1.4.4
```

No daemon. No background indexer. The repository is a directory
of segment files plus a small SQLite-shaped index; you can
`tar` it up and move it to another host without breaking
anything.

## License

BSD-3-Clause — see
[LICENSE](https://github.com/borgbackup/borg/blob/master/LICENSE).
Permissive; embedding the binary in a paid product or shipping
it inside a Docker image is fine.

## One Concrete Example

```bash
# 1. one-time: initialise an encrypted repository on a USB disk
borg init --encryption=repokey-blake2 /Volumes/Backup/borgrepo

# 2. nightly snapshot of $HOME, excluding the loud directories
borg create \
  --stats --progress --compression zstd,6 \
  --exclude '*/node_modules' --exclude '*/.cache' \
  --exclude '*/Library/Caches' --exclude '*/.venv' \
  /Volumes/Backup/borgrepo::"home-{hostname}-{now:%Y-%m-%dT%H:%M}" \
  ~

# 3. push to a remote repo over SSH (borg installed on both ends)
borg create --compression zstd,6 \
  ssh://backup@nas.lan:22/srv/borg/laptop::"{hostname}-{now}" \
  ~/code ~/Documents

# 4. retention policy: 7 daily, 4 weekly, 12 monthly
borg prune --list --keep-daily 7 --keep-weekly 4 --keep-monthly 12 \
  /Volumes/Backup/borgrepo

# 5. compact (actually frees disk after prune; v1.4 requires this step)
borg compact /Volumes/Backup/borgrepo

# 6. browse a snapshot like a normal filesystem
mkdir /tmp/snap
borg mount /Volumes/Backup/borgrepo::home-laptop-2026-04-30T02:00 /tmp/snap
ls /tmp/snap/code/myproject/    # cp anything you need
borg umount /tmp/snap

# 7. extract one file to $PWD without mounting
borg extract --list /Volumes/Backup/borgrepo::home-laptop-2026-04-30T02:00 \
  home/me/code/myproject/config.toml

# 8. verify repository integrity (checks all chunk hashes)
borg check --verify-data /Volumes/Backup/borgrepo

# 9. see what changed between two snapshots
borg diff /Volumes/Backup/borgrepo::home-laptop-2026-04-29T02:00 \
                                    home-laptop-2026-04-30T02:00
```

## Niche It Fills

**Snapshot-style backup with deduplication that survives moving
files around.** Because chunking is content-defined (rolling
hash, not fixed boundaries), renaming a 5 GB video directory or
moving a project from `~/code` to `~/work` does not cost a
single extra byte in the repo — the chunks are already there
under the same content hash. This is the property that
`rsync --link-dest`, `tar` snapshots, and most filesystem-level
snapshot tools (`zfs send`, `btrfs send`) do not give you across
arbitrary moves and renames. It is the same property that makes
restic and Kopia popular, and Borg is the older, more battle-
tested member of that family.

## Why use it

Three things `borg` does that picking a folder-mirror tool does
not, that explain its survival in the era of `restic` /
`kopia` / `duplicacy`:

1. **Authenticated encryption is the default, not a flag.** With
   `--encryption=repokey-blake2`, every chunk is encrypted with
   AES-256-CTR + BLAKE2b-MAC before it leaves the host; the key
   is itself wrapped with your passphrase and stored inside the
   repo (or with `keyfile` mode, separately). The backup target
   — whether it's `rsync.net`, a coworker's NAS, or a USB stick
   that might walk away — sees opaque bytes only. No "remember
   to also enable encryption" footgun.
2. **One repo, many machines, full dedup across all of them.**
   Point three laptops at the same SSH-accessible repo with
   different `BORG_PASSPHRASE`s and they all benefit from
   cross-host deduplication: the 4 GB Xcode toolchain that
   exists on every dev machine is stored once. `borg list` shows
   all snapshots from all hosts; per-snapshot prefixes
   (`{hostname}-…`) make per-host pruning trivial.
3. **`borg mount` makes restore a `cp`, not a procedure.** Most
   backup tools have a "restore" verb that reconstructs into a
   target directory. Borg just FUSE-mounts any snapshot at any
   path, so restore becomes the same `ls`/`cp`/`grep` you
   already know — including grepping across snapshots when you
   are trying to remember "which version of `config.toml` had
   the working setting".

For an LLM-CLI workflow, `borg create … ~/.config/agent` before
a destructive `agent rm -rf cache/` is a one-line "checkpoint
this state cheaply"; `borg diff` between two snapshots is a
machine-readable record of what an autonomous step changed on
disk outside its own working tree.

## Vs Already Cataloged

- **Vs [`restic`](../restic/):** Restic is the closest peer and
  the more popular modern choice. Both do content-defined
  chunking, both encrypt by default, both speak many backends.
  Tradeoffs: restic speaks S3/GCS/Azure/Backblaze/SFTP/REST
  natively (no helper on the remote), Borg requires a `borg`
  binary at the SSH target or uses a local-disk repo; restic
  uses a single global lock for `prune`, Borg's `prune +
  compact` is a two-step but more granular; Borg's `borg mount`
  is FUSE-native, restic's `restic mount` exists but is younger.
  Pick restic for cloud-object-store targets and a single static
  Go binary; pick Borg for SSH+disk targets, FUSE-first restore
  workflows, and a longer track record (since 2010, forked from
  Attic).
- **Vs [`rclone`](../rclone/):** rclone is `rsync` for cloud
  storage — it mirrors directories to/from 70+ providers. It
  has a `crypt` overlay and a `--backup-dir` flag, but it does
  not do snapshot-style retention or content-defined dedup.
  Pick rclone for "make this S3 bucket look like that local
  folder"; pick Borg for "I want 90 daily snapshots and the
  repo to fit on one disk".
- **Vs `tar` + `rsync --link-dest` (not cataloged):** Classic
  Unix snapshot pattern; works, costs no install. But hardlink-
  based dedup is per-file-identity, so a renamed file is a full
  copy, an edited file is a full copy, and you cannot encrypt
  without a separate layer. Pick the classic stack only if you
  cannot install a binary on the backup host; otherwise Borg
  wins on space and on encryption.
- **Vs filesystem snapshots (`zfs`, `btrfs`, APFS):** FS
  snapshots are O(1) creation and great for local rollback
  ("undo last 30s of edits") but live in the same pool as the
  data — a disk failure or an `rm -rf` of the snapshot dir
  loses everything. Borg targets a *separate* repository,
  optionally on a different host, with cryptographic
  integrity. They are complements, not substitutes: snapshots
  for "oops just now", Borg for "the laptop was stolen".

## Caveats

- **`borg compact` is a separate step in 1.x.** After `borg
  prune` removes old snapshots, freed space is not returned to
  the filesystem until `borg compact` runs. Forget this and a
  "pruned" repo keeps growing. (Borg 2.x changes this; 1.4.x is
  the current stable line.)
- **Repository lock contention.** Only one writer at a time per
  repo; concurrent `borg create` from two hosts fails fast with
  a stale-lock error rather than corrupting state. For multi-
  host scheduling, stagger crons or use `--lock-wait`.
- **Borg version on both ends must match (major+minor).** A
  laptop running `borg 1.4` cannot push to an SSH target with
  `borg 1.2` — you'll get a protocol mismatch. Plan host
  upgrades together, or pin both to the same minor.
- **Passphrase loss is data loss.** With `repokey` encryption,
  the wrapped key lives in the repo so you only need the
  passphrase; with `keyfile`, you need both. Lose either and the
  repo is unrecoverable noise. Print/escrow your key (`borg key
  export`) somewhere out-of-band.
- **Initial backup is slow; subsequent ones are fast.** First
  `borg create` on a 500 GB home dir takes hours because every
  byte must be hashed, chunked, compressed, encrypted, and
  written. Daily incrementals usually finish in seconds-to-
  minutes. Don't benchmark on the first run.
- **Mac extended attributes / ACLs round-trip imperfectly.** As
  of 1.4.x, BSD/macOS xattrs and ACLs are preserved in most
  cases, but Finder tags and some Spotlight metadata may not
  survive restore. Test-restore a `.app` bundle before trusting
  Borg as your sole macOS backup.
