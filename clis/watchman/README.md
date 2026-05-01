# watchman

## What it does
A persistent file-watching service: `watchman` is a long-running daemon (one per user, started lazily by the CLI) that uses the kernel-level fs notification API on the host (FSEvents on macOS, inotify on Linux, ReadDirectoryChangesW on Windows) to maintain a coalesced, deduplicated index of every change under each *watched root*, and answers queries like "give me every file under `src/` whose mtime advanced since clock `c:1714000000:42`" in O(changes) instead of O(tree). Tools register a watch (`watchman watch /repo`), then either subscribe (`watchman -- trigger /repo build '*.ts' -- npm run build`) or poll with a since-clock — so a 200k-file repo gets near-instant change notifications without ever re-walking the tree.

## Why it's interesting
Different shape from `entr` / `fswatch` / `chokidar` / `nodemon` (per-process file watchers — they re-walk and hold a watcher per file descriptor, which falls over above ~100k files and dies when the process exits) and from polling tools like `watchexec` (great for one-shot dev loops, but still per-invocation). `watchman` is the *service* layer underneath build systems — Buck2, Mercurial's `hg status`, Jest's `--watch`, Metro bundler, and Sapling all consume it precisely because the daemon amortizes the kernel watcher cost across every consumer on the box. Choose it when you have one giant repo and many tools that all want change notifications; do **not** choose it for a 50-file project where `entr` or `watchexec` is one binary and zero state.

## Niche category
File-watching daemon — shared kernel-fs-notification index queryable by many clients.

## Repo
https://github.com/facebook/watchman

## Version pinned
`v2026.04.27.00`

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
brew install watchman
# or build from source per https://facebook.github.io/watchman/docs/install
# prebuilt binaries (macOS / Linux / Windows) are attached to the v2026.04.27.00 release
```

## Usage examples
```sh
# Register a watch on a repo (starts the daemon if not running)
watchman watch ~/code/myrepo

# What is being watched right now?
watchman watch-list

# Get a clock token, do work, then ask "what changed since"
CLOCK=$(watchman clock ~/code/myrepo | jq -r .clock)
# ... edit some files ...
watchman -j <<EOF
["query", "$HOME/code/myrepo", {
  "since": "$CLOCK",
  "expression": ["allof", ["match", "*.ts"], ["type", "f"]],
  "fields": ["name", "mtime_ms"]
}]
EOF

# Run a command whenever a TS file changes (long-lived trigger)
watchman -- trigger ~/code/myrepo build '*.ts' -- npm run build

# Stop watching and shut the daemon down cleanly
watchman watch-del ~/code/myrepo
watchman shutdown-server
```
