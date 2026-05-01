# wasmtime

## What it does
A standalone, lightweight WebAssembly runtime built by the Bytecode Alliance. It executes `.wasm` and WASI modules outside the browser with a focus on speed, security (sandboxed by default), and standards compliance.

## Why it's interesting
Reference implementation for the WASI standard and one of the most production-hardened WASM runtimes. Useful as a sandbox for plugin systems, edge compute, and as a portable execution target for AI tooling that wants to ship language-agnostic, sandboxed extensions without spinning up containers.

## Niche category
WebAssembly tooling — runtime / WASI host

## Repo
https://github.com/bytecodealliance/wasmtime

## Version pinned
`v44.0.1`

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
curl https://wasmtime.dev/install.sh -sSf | bash
# or
brew install wasmtime
```

## Usage examples
```sh
# Run a WASI module
wasmtime run hello.wasm

# Pass args and preopen a directory for WASI filesystem access
wasmtime run --dir=. compute.wasm -- input.txt
```
