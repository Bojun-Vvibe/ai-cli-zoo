# reflex

> **A minimal file-watcher that runs commands when files change — `entr`-class ergonomics with regex includes/excludes, named groups, and a config file for many parallel watches**
> — by Caleb Spare. A single Go binary that watches a directory
> tree and runs commands in response to file events, with first-class
> support for *substituting the changed path* into the command
> (`{}`), *batching* concurrent events, and running *multiple*
> independent watches from one config file. Pinned to **v0.3.1**
> ([LICENSE.txt](https://github.com/cespare/reflex/blob/master/LICENSE.txt),
> Apache-2.0).

Source: <https://github.com/cespare/reflex>

## TL;DR

`reflex` is what you reach for when [`entr`](../entr/) is *almost*
right but you want three things `entr` doesn't give you cleanly:
(1) **regex** include/exclude rather than a `find | entr` pre-filter,
(2) the path of the **specific changed file** substituted into the
command (`{}` placeholder, like `xargs -I`), and (3) a **config
file** so you can declare half a dozen independent watches —
"run tests on `*_test.go`", "rebuild the docs on `*.md`", "lint
on `*.{ts,tsx}`" — once and start them all with a single
`reflex -c reflex.conf`. Each rule runs in its own goroutine
with its own debounce window, so a save that touches twenty
files triggers each rule at most once.

The other thing `reflex` gets right is killing the previous
invocation when a new event arrives mid-run (`-s` /
`--start-service`): essential for long-running dev servers
where you want a fresh `go run ./cmd/api` on every save and the
old process to die cleanly first. `entr -r` does this too, but
`reflex`'s SIGINT-then-SIGKILL ladder is gentler and survives
processes that ignore the first signal.

## Install

```bash
# Homebrew (macOS / Linux)
brew install reflex

# Go
go install github.com/cespare/reflex@latest

# Release binary
curl -L https://github.com/cespare/reflex/releases/download/v0.3.1/reflex_linux_amd64.tar.gz \
  | tar -xz -C /usr/local/bin reflex

# verify
reflex --version    # reflex 0.3.1
```

No daemon, no shell hook. The watcher uses `fsnotify` under the
hood, so on Linux the kernel inotify limit applies — bump
`fs.inotify.max_user_watches` if you are watching a `node_modules`
tree (or, better, exclude it; see below).

## Usage

```bash
# 1) Re-run the Go test suite for the package containing the
#    file you just saved. {} is the changed path; %x extracts
#    its directory.
reflex -r '\.go$' -- sh -c 'go test ./$(dirname {})'

# 2) Long-running dev server: kill the old process before
#    starting the new one when any *.go file changes, but
#    ignore the build artefacts directory.
reflex -s -r '\.go$' -R '^bin/' -- go run ./cmd/api

# 3) Many watches from one file (reflex.conf):
#       -r '\.go$'           -- go test ./...
#       -r '\.md$'           -- mdbook build
#       -r '\.(ts|tsx|css)$' -- npm run lint
#    then:
reflex -c reflex.conf
```

## Niche & tradeoffs

`reflex` lives in the same neighbourhood as
[`entr`](../entr/), [`watchexec`](../watchexec/),
[`fswatch`](../fswatch/), and [`watchman`](../watchman/), and the
right one to pick depends entirely on which axis you care about
most. `entr` is the smallest and most Unix-philosophy of the
group: you pipe a file list into it, it watches them, and that's
it — but you cannot easily say "watch this whole tree, except
`node_modules` and `dist`, and re-run the command with the
changed path substituted in." `watchexec` is closer in spirit
to `reflex` and arguably wins on raw features (built-in
`.gitignore` honouring, environment-variable injection of the
event list, a richer signal protocol) but its CLI surface is
larger and its config story is "lots of flags" rather than
"one declarative file." `watchman` is a Facebook-grade
*service* with a query language and persistent state — overkill
for a dev loop, indispensable for a build system.

The tradeoffs to internalise: (1) `reflex` is unmaintained-ish
— the last tagged release is `v0.3.1` and issues sit for a
long time; that is fine for a tool this small (the surface area
is genuinely done) but means you should not expect new features.
For an actively-developed alternative pick `watchexec`.
(2) On macOS the `fsnotify` backend uses `FSEvents`, which
coalesces events at ~10ms granularity; if you need
file-by-file fidelity below that, you'll see batches rather
than individual events. (3) The regex flavour is Go's `regexp`
(RE2) — no lookarounds, no backreferences. For the file-glob
patterns you actually want, that is fine; for clever exclude
patterns you may need to chain `-R`s.

The right mental model is "**`entr` with regex and a config
file, in one Go binary**." If your dev loop is a single command
on a single file pattern, stay with `entr` — it is smaller and
more obviously correct. The moment you have *two or more*
parallel watches with different commands and different file
patterns, or you want `{}` substitution, `reflex` pays for
itself on day one. For richer pipelines (env-var event lists,
Git-aware ignore, JSON output) graduate to
[`watchexec`](../watchexec/); for build-system-grade watching
with a query language, [`watchman`](../watchman/).
