# spin

## What it does
A developer tool for building and running serverless applications compiled to WebAssembly. Spin handles the scaffolding, local dev loop, and production execution of HTTP, Redis, and cron-triggered components written in Rust, Go, JavaScript, Python, and other WASM-targeting languages.

## Why it's interesting
Brings a serverless / FaaS programming model to WebAssembly with sub-millisecond cold starts. A practical way to ship polyglot, sandboxed services without containers — the runtime model is much lighter than a typical k8s-based serverless platform, which makes it attractive for edge and embedded scenarios.

## Niche category
Serverless / edge — WebAssembly application framework

## Repo
https://github.com/spinframework/spin

## Version pinned
`v4.0.0`

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
curl -fsSL https://wasmtime.dev/install.sh | bash  # optional, for WASM tooling
brew install fermyon/tap/spin
# or download release binary from the repo
```

## Usage examples
```sh
# Scaffold a new HTTP component (Rust)
spin new -t http-rust hello-spin

# Build and run locally
cd hello-spin && spin build && spin up
```
