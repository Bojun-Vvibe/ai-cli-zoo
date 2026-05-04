# bkt

> **Subprocess cache for the command line.**
> A single Rust binary (`bkt`) that wraps any shell command, hashes
> the argv + working directory + a configurable subset of env vars,
> and either replays the cached stdout/stderr/exit-code from disk or
> runs the command and stores the result with a TTL. Pinned to
> **v0.8.1** (released 2024-07-21, SPDX: `Apache-2.0`,
> [LICENSE](https://github.com/dimo414/bkt/blob/master/LICENSE)).

Source: <https://github.com/dimo414/bkt>

## TL;DR

`bkt --ttl=60s -- kubectl get pods` runs `kubectl get pods` the
first time and stores the output for 60 seconds; the next ten
invocations in that minute return instantly from the cache. Pair it
with shell prompts, status bars, and `tmux` status-line scripts to
turn slow upstream commands (cloud APIs, `kubectl`, `aws`, `gh`,
`curl` to a flaky service) into effectively-free reads without
writing a custom caching layer.

## Install

```sh
# Cargo:
cargo install bkt --version 0.8.1

# Homebrew:
brew install bkt

# Verify:
bkt --version    # 0.8.1
```

## License

Apache-2.0 — unrestricted, includes a patent grant. Safe to embed
in shell prompts, internal CLI wrappers, dotfiles repos, or vendor
into a closed-source operator tool that needs a generic subprocess
cache.

## Primary use case

A `tmux` status line or `starship`-style prompt that wants to show
"current AWS account / current k8s context / current GitHub PR
count" but `aws sts get-caller-identity` takes 800 ms, `kubectl
config current-context` takes 50 ms, and `gh pr status` takes 1.2 s
— and the prompt redraws every keystroke. Wrap each one in
`bkt --ttl=30s --` and the prompt stays interactive while the
underlying values still refresh every half-minute.

Other natural fits: cron jobs that poll a slow API, CI scripts that
re-invoke `terraform output` dozens of times across stages, and any
"watch this command but don't hammer the upstream" loop.

## What it competes with

- **Hand-rolled file caches** (`if [ -f /tmp/foo ] && ...`) —
  every shell developer has written one. `bkt` adds proper TTL,
  per-argv keying, env-var scoping, stale-while-revalidate (`--stale`),
  and concurrent-call coalescing. The build-vs-buy break-even is
  about 30 lines of bash.
- **`memoize` / shell-function memoization libraries** — in-memory
  only, lost on shell exit. `bkt` persists across shells and
  processes (cache lives at `$XDG_CACHE_HOME/bkt`).
- **HTTP-level caches** (`curl --cache`, `varnish`, `mitmproxy`
  with caching) — only help when the slow command is HTTP. `bkt`
  is process-agnostic; it works on `kubectl`, `aws`, `terraform`,
  any binary.
- **`watch -n 30 cmd > file; cat file`** — an active poller. `bkt`
  is lazy — it only re-runs when something actually asks for the
  value *and* the TTL has expired, so a prompt that nobody redraws
  costs zero upstream calls.
- **[`hyperfine`](../hyperfine/)** — different problem entirely
  (benchmarking). Mentioned only because both are "wrap a
  subprocess and do something clever with the result."

## Why include

The catalog already covers shell prompts (`starship`, `oh-my-posh`),
status bars (`wtfutil`), and benchmarking (`hyperfine`), but no
generic **subprocess result cache**. `bkt` fills that gap with a
single Rust binary, a 5-flag CLI, and an Apache-2.0 license — the
right tool any time a script calls the same slow command more than
once in a TTL window.
