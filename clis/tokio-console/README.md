# tokio-console

## What it does
A live, htop-style debugger for async Rust programs built on `tokio`. The instrumented binary exports per-task / per-resource telemetry over a gRPC endpoint (default `127.0.0.1:6669`) via the `console-subscriber` crate; the `tokio-console` TUI then connects to that endpoint and shows every spawned task with its state, polls, busy / idle / scheduled time, wakers held, and the source location it was spawned from. It also has dedicated views for resources (mutexes, semaphores, oneshot channels) and async ops (lock acquires, send/recv) so you can see *which* task is blocking on *which* resource right now, not just "the program feels slow".

## Why it's interesting
Different shape from `tokio-metrics` (numerical aggregates only — no per-task drilldown), `flamegraph` / `samply` (CPU sampling — invisible to async tasks parked on a waker), and `tracing` + `tracing-subscriber` text logs (after-the-fact, no live state). `tokio-console` is the only tool that surfaces the *runtime-level* picture (tasks the scheduler currently knows about + their wake history) in real time, which is the only useful view when the bug is "a task never gets polled again" or "10 000 tasks are stuck on a `Mutex` held by one slow task".

## Niche category
Async-runtime live debugger — gRPC-attached TUI for `tokio` task / resource state.

## Repo
https://github.com/tokio-rs/console

## Version pinned
`tokio-console-v0.1.14` (CLI binary; pair with `console-subscriber v0.5.0` in the target binary)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
cargo install --locked tokio-console
# or
brew install tokio-console
```

## Usage examples
```sh
# 1. In the target Rust binary, add the subscriber and start it:
#    [dependencies]
#    console-subscriber = "0.5"
#    tokio = { version = "1", features = ["full", "tracing"] }
#
#    fn main() {
#        console_subscriber::init();
#        // ... rest of your tokio runtime ...
#    }
#
#    Build with the tokio_unstable cfg flag (required for task tracing):
RUSTFLAGS="--cfg tokio_unstable" cargo run --release

# 2. In another terminal, attach the TUI to the default endpoint:
tokio-console

# 3. Or attach to a remote / non-default endpoint:
tokio-console http://10.0.0.7:6669

# Inside the TUI:
#   t = tasks view, r = resources view, a = async ops view
#   Enter on a row drills into per-task poll histogram + wake history
#   q exits
```
