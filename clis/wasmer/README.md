# wasmer

## What it does
Universal WebAssembly runtime that executes `.wasm` modules outside the browser via multiple pluggable backends (Cranelift, LLVM, Singlepass) and supports both WASI and the proprietary WASIX extension for richer POSIX-like syscalls (threads, sockets, fork/exec). Ships with `wasmer run` for direct execution and `wasmer package` / `wasmer publish` for the wasmer.io registry of redistributable WASM apps.

## Why it's interesting
Orthogonal alternative to wasmtime: wasmer leans into "WASM as a portable application format" with its own registry and the WASIX extension that closes the gap between WASM and a real Unix process model, while wasmtime sticks closer to upstream WASI. Useful when you want to ship a single `.wasm` artifact that runs threaded network code on macOS, Linux, Windows, and inside other host languages via the `wasmer` embeddings.

## Niche category
WebAssembly tooling — runtime / WASIX host + package registry client

## Repo
https://github.com/wasmerio/wasmer

## Version pinned
`v7.1.0`

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
curl https://get.wasmer.io -sSfL | sh
# or
brew install wasmer
```

## Usage examples
```sh
# Run a local WASI module
wasmer run hello.wasm

# Run a published package straight from the registry
wasmer run wasmer/python -- -c "print(2**16)"

# Pick a compiler backend explicitly
wasmer run --llvm compute.wasm
```
