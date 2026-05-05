# wasm-tools

> **The Bytecode Alliance's swiss-army knife for WebAssembly** —
> one Rust binary that parses, validates, prints, assembles,
> componentises, strips, mutates, fuzzes, and diffs `.wasm` /
> `.wat` files at every level of the WebAssembly + Component Model
> stack. Pinned to **v1.248.0** (released 2026-04-28,
> [`gh api repos/bytecodealliance/wasm-tools/releases/latest`](https://github.com/bytecodealliance/wasm-tools/releases/latest),
> [LICENSE-APACHE](https://github.com/bytecodealliance/wasm-tools/blob/main/LICENSE-APACHE),
> Apache-2.0 WITH LLVM-exception, also available as Apache-2.0 / MIT).

Source: <https://github.com/bytecodealliance/wasm-tools>

## TL;DR

If you ship Wasm — server-side (Wasmtime, WasmEdge, Spin), browser
(via `wasm-bindgen`), edge (Fastly, Cloudflare Workers), or
plug-in surfaces (Envoy, Istio, Zellij, Tree-sitter) — at some point
you need to inspect the bytes: "is this module valid against the
current proposals?", "what imports does it actually use?", "what's
the disassembly of function 42?", "wrap this core module as a
Component-Model component", "strip the debug section before
shipping". `wasm-tools` is the canonical answer, maintained by the
same Bytecode Alliance group that ships Wasmtime, and tracks the
proposal landscape in lockstep (GC, exception-handling, threads,
component-model, memory64). Subcommands form a coherent verb set:
`parse` / `print` (binary ↔ text), `validate` (proposal-aware),
`dump` (raw section walker), `objdump` (linker-style symbol view),
`strip` (remove sections), `shrink` / `mutate` / `smith` (testcase
shrinking + structured fuzz inputs), `compose` / `component new` /
`component wit` (Component Model toolchain), `metadata add` /
`metadata show` (registry-style metadata).

## Install

```bash
# Cargo (Rust toolchain on any platform)
cargo install --locked wasm-tools

# Homebrew (macOS / Linux)
brew install wasm-tools

# Pre-built binary from a release
curl -L \
  https://github.com/bytecodealliance/wasm-tools/releases/download/v1.248.0/wasm-tools-1.248.0-x86_64-linux.tar.gz \
  | tar xz && sudo mv wasm-tools-1.248.0-x86_64-linux/wasm-tools /usr/local/bin/

# verify
wasm-tools --version    # wasm-tools 1.248.0
```

## Representative examples

```bash
# 1. Disassemble a .wasm to readable .wat
wasm-tools print foo.wasm > foo.wat

# 2. Re-assemble (round-trip)
wasm-tools parse foo.wat -o foo.wasm

# 3. Validate against the latest stable proposals
wasm-tools validate foo.wasm

# 4. Validate enabling specific in-flight proposals
wasm-tools validate --features=gc,exceptions,threads foo.wasm

# 5. Strip debug sections to shrink the shipped artifact
wasm-tools strip foo.wasm -o foo.stripped.wasm

# 6. Wrap a core module as a Component-Model component
wasm-tools component new core.wasm -o my.component.wasm \
  --adapt wasi_snapshot_preview1=adapter.wasm

# 7. Show the WIT (interface definition) embedded in a component
wasm-tools component wit my.component.wasm

# 8. Dump every section's offset + size for size-budget analysis
wasm-tools dump foo.wasm | head -50
```

## When to use vs. alternatives

- Pick **wasm-tools** for any Wasm bytecode operation that is *not*
  "compile my source to Wasm" — it is the layer below the language
  toolchain, the equivalent of `binutils` (`objdump` / `objcopy` /
  `strip` / `nm`) for the Wasm world, and tracks proposal flags as
  they ship in Wasmtime.
- Pick [`binaryen`](https://github.com/WebAssembly/binaryen) (`wasm-opt`,
  `wasm-as`, `wasm-dis`) when the goal is *optimisation* (LICM,
  inlining, DCE, size-optimised passes) — wasm-tools deliberately
  doesn't optimise. The standard pipeline is
  `wasm-tools strip | wasm-opt -Oz`.
- Pick `wabt` (WebAssembly Binary Toolkit, the historical reference
  implementation in C++ — `wasm2wat` / `wat2wasm` / `wasm-objdump`)
  when you specifically need wabt's interpreter or are pinned to it
  by tradition; otherwise prefer wasm-tools, which is more actively
  maintained, faster, and proposal-current.
- Pick [`wasmer`](../wasmer/) / [`wasmtime`](https://wasmtime.dev/) /
  [`wasmedge`](https://wasmedge.org/) when you want to *run* the
  Wasm — wasm-tools only inspects and transforms.
- Pick [`oras`](../oras/) for the orthogonal concern: pushing /
  pulling the resulting `.wasm` artifact to/from an OCI registry
  (Spin, Fermyon, and the WASI SDK distribution all use this).
