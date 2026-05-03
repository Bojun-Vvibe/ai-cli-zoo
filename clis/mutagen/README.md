# mutagen

> **A high-performance, real-time, two-way file
> synchronization daemon** built specifically for
> the "edit locally, run remotely" loop —
> `mutagen sync create --name=proj ~/code/proj
> ssh://box/home/me/code/proj` and every save
> propagates in milliseconds, with conflict
> resolution, ignore filters, file-mode
> normalization across OSes, and a separate
> `mutagen forward` subsystem for TCP / Unix-
> socket tunnels in the same process. Pinned to
> **v0.18.1**
> ([LICENSE](https://github.com/mutagen-io/mutagen/blob/master/LICENSE),
> MIT; SSPL components opt-in via the `--sspl`
> build flag).

Source: <https://github.com/mutagen-io/mutagen>

## TL;DR

You edit on your laptop. The build, the
container, the GPU, the database, the staging
cluster — they live somewhere else. The classic
options are: (a) edit over SSH/SFTP and tolerate
the latency, (b) `rsync` on save with a watcher
and tolerate one-way drift, (c) mount the remote
over `sshfs` and tolerate the IO model, (d) push
through git on every change and tolerate the
ceremony. `mutagen` is the answer that says
"none of those — just keep two trees identical
in real time, both directions, and tell me when
you have a true conflict." It runs as a per-user
daemon, watches with native filesystem events
(FSEvents / inotify / ReadDirectoryChangesW),
ships only the deltas, normalizes file modes and
symlinks across mismatched OSes, and gives you a
single `mutagen sync list` to see lag, conflicts,
and throughput per session.

## Install

```bash
# Homebrew tap
brew install mutagen-io/mutagen/mutagen

# Pre-built binary from the release page
# (darwin-arm64 / darwin-amd64 / linux-* / windows-*)
curl -L -o /tmp/mutagen.tgz \
  https://github.com/mutagen-io/mutagen/releases/download/v0.18.1/mutagen_darwin_arm64_v0.18.1.tar.gz
tar -xzf /tmp/mutagen.tgz -C /usr/local/bin mutagen mutagen-agent
chmod +x /usr/local/bin/mutagen /usr/local/bin/mutagen-agent

# Verify
mutagen version    # 0.18.1
```

Single static Go binary plus a helper
`mutagen-agent` that the daemon copies to the
remote over SSH on first sync — no manual
install on the remote side. The daemon
auto-starts per-user on first command.

## Use it for

```bash
# 1) Two-way real-time sync to a remote box
mutagen sync create --name=proj \
  ~/code/proj \
  ssh://devbox.example.com/home/me/code/proj
# Every save on either side propagates within
# milliseconds. `mutagen sync list` shows lag.

# 2) Sync into a running container
mutagen sync create --name=app \
  ~/code/app \
  docker://my-container/workspace

# 3) Ignore generated / large / vendored paths
mutagen sync create --name=proj \
  --ignore='node_modules' --ignore='/dist' \
  --ignore='.venv' --ignore='target' \
  ~/code/proj ssh://box/home/me/code/proj

# 4) Permission / ownership normalization
#    (so a 0644 on macOS arrives as 0644 on Linux,
#     not 0755 from a confused sshfs mount)
mutagen sync create --name=proj \
  --default-file-mode=0644 --default-directory-mode=0755 \
  --default-owner=me --default-group=me \
  ~/code/proj ssh://box/home/me/code/proj

# 5) Conflict-resolution policy
#    (default = manual; alpha-wins / beta-wins
#     for read-only mirroring scenarios)
mutagen sync create --name=mirror \
  --sync-mode=one-way-replica \
  ~/snapshot ssh://archive/srv/snapshot

# 6) Forward a TCP port through the same SSH
#    transport (no second tunnel to manage)
mutagen forward create --name=pg \
  tcp:localhost:5432 ssh://box:tcp:localhost:5432
mutagen forward create --name=redis \
  tcp:localhost:6379 ssh://box:tcp:localhost:6379

# 7) Pause / resume / flush a session
mutagen sync pause proj
mutagen sync resume proj
mutagen sync flush proj    # block until in-sync

# 8) Project-level orchestration via mutagen.yml
#    `mutagen project start` brings up every sync
#    + forward defined in the file, `stop`
#    tears them all down.
cat > mutagen.yml <<'YAML'
sync:
  defaults:
    ignore:
      paths: [node_modules, .venv, dist, target]
  code:
    alpha: "."
    beta: "ssh://devbox/home/me/proj"
forward:
  pg:
    source: "tcp:localhost:5432"
    destination: "ssh://devbox:tcp:localhost:5432"
YAML
mutagen project start
mutagen project list
mutagen project terminate
```

The flag set is large because the problem is
gnarly (cross-OS mode mapping, symlink modes,
watch-vs-poll, scan throttling, conflict
policy), but the *common* invocation is one
line.

## Why include it in a CLI catalog

1. **It is the canonical "edit-locally,
   run-remotely" sync tool.** Not file-transfer
   (`rsync`), not network filesystem (`sshfs`),
   not version control (`git`) — a *third
   category*: live two-way mirror with conflict
   handling. Any workflow that involves a beefy
   remote machine (GPU box, dev container,
   staging VM) and a local editor benefits
   immediately.
2. **The forward subsystem removes a whole
   category of `ssh -L` plumbing.** Postgres,
   Redis, MinIO, Jupyter, an internal HTTP
   service — declare them in `mutagen.yml`
   alongside the sync, `mutagen project start`
   brings everything up, single `terminate`
   takes it down. No tunnel-manager script, no
   stale background `ssh` processes.
3. **Cross-OS file-mode normalization is rare
   and important.** Edit on macOS, run in a
   Linux container, and modes / ownership /
   symlink semantics *will* drift. Mutagen has
   first-class config for this; most alternatives
   just hand you the raw inode and wish you luck.

For an LLM-CLI workflow, `mutagen` is the "agent
runs on the GPU box, but I review and edit on my
laptop" piece: bidirectional sync means the
agent's edits land on your local tree
immediately (so your editor / LSP / git client
see them) and your manual fixups land on the
remote box immediately (so the next agent step
sees them). Far cleaner than juggling
`rsync --delete` cron + a one-way watcher.

## Vs Already Cataloged

- **Vs [`rclone`](../rclone/):** different axis.
  `rclone` is a *one-shot* (or scheduled) cloud-
  storage transfer tool — buckets, blobs, drive
  APIs, encryption-on-the-wire. Mutagen is a
  *long-lived* two-way local-tree-to-local-tree
  daemon over SSH/Docker/local. Different
  transports, different temporal model, different
  use case. Use `rclone` to push a build artifact
  to S3; use `mutagen` to keep your editor in
  sync with a dev container.
- **Vs [`syncthing`](../syncthing/):** sibling
  but different shape. Syncthing is a *peer-to-
  peer* mesh designed for personal devices
  (laptop ↔ phone ↔ NAS) over its own
  discovery / relay protocol, with a web UI and
  an always-on "every device knows about every
  other" model. Mutagen is *point-to-point* over
  SSH/Docker, designed for one developer's
  workflow with a remote machine, with a CLI
  and per-project config. Pick syncthing for
  household / personal-device sync; pick mutagen
  for dev-loop sync.
- **Vs `unison`:** direct ancestor. Unison is the
  classic OCaml two-way sync; mutagen is the
  modern Go reimagining with native FS-event
  watching, container support, project config,
  and TCP forwarding in the same daemon. If
  you're starting fresh in 2026, mutagen.
- **Vs `sshfs` / `nfs` / `9p`:** orthogonal.
  Network filesystems present the remote tree
  *as* a mount; every editor read is a network
  round-trip (and editors that scan trees on
  open get punished). Mutagen keeps a *real
  local copy* and synchronizes it; editor reads
  are local-disk speed.
- **Vs IDE remote-development modes** (e.g.
  remote-SSH plugins): orthogonal — those move
  the *editor* to the remote and present a thin
  client locally. Mutagen keeps the editor
  local and the files local-and-remote. Pick
  by where your tooling (LSP, formatters,
  shell, git) is already configured.

## Caveats

- **The daemon is per-user, not per-project.**
  Sessions outlive shells; `mutagen sync list`
  is the source of truth, and stale sessions
  from old branches *will* keep replicating in
  the background. Get into the habit of
  `mutagen project terminate` (or `sync
  terminate <name>`) when you're done.
- **Initial scan of a huge tree is expensive.**
  First sync of a million-file repo (think a
  monorepo with `node_modules` not ignored) will
  hash everything end-to-end. Always set
  `--ignore` for generated dirs *before* the
  first sync, not after.
- **Conflicts surface as a session-level
  alert, not a per-file prompt.** `mutagen sync
  list` reports `Conflicts: N`; you resolve by
  picking a side (`mutagen sync flush --skip-
  wait` after deleting the loser, or
  `--sync-mode` change). There is no built-in
  three-way merge — that's still your editor's
  job.
- **SSPL components are bundled in default
  release builds from v0.17 onward.** The MIT
  core is unaffected, but if your distribution
  policy disallows SSPL artifacts, build from
  source without `--sspl`. The
  [LICENSE](https://github.com/mutagen-io/mutagen/blob/master/LICENSE)
  file documents this clearly.
- **Forward sessions share the SSH transport
  with sync sessions.** A flaky link drops both
  at once; `mutagen daemon stop && mutagen sync
  list` reconnects them. There's no separate
  retry loop per forward.
- **MIT license** for the core (with opt-in
  SSPL extensions) — permissive; safe to ship
  inside proprietary toolchains and dev-env
  installers with attribution. Check
  [`LICENSE`](https://github.com/mutagen-io/mutagen/blob/master/LICENSE)
  for the SSPL boundary if you're rebundling.
