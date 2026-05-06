# wasmedge

> Snapshot date: 2026-05. Upstream: <https://github.com/WasmEdge/WasmEdge>

**A CNCF-graduated, cloud-native WebAssembly runtime with a
plugin-driven host-function model — runs `.wasm` modules from the
shell, embeds in Go / Rust / Node / Python / C++ apps, and serves
as the OCI-compatible WASM engine for `crun` / `containerd` /
Docker so a Wasm workload schedules in Kubernetes the same way a
container does.**
WasmEdge ships one CLI (`wasmedge`, plus `wasmedge compile` for
AOT), a daemon-free interpreter + AOT compiler, and a plugin
catalog (WASI-NN for ONNX / GGML / PyTorch / TF-Lite inference,
WASI-Crypto, WASI-HTTP, WasmEdge-RDBMS, WasmEdge-TensorFlow) that
extends the sandbox with capability-scoped host functions instead
of the all-or-nothing `--allow-all` posture small Wasm runtimes
default to.

## Repo + version + license

- Repo: <https://github.com/WasmEdge/WasmEdge>
- Latest release: **`0.16.2`** (2026-04-15)
- License: **Apache-2.0** —
  <https://github.com/WasmEdge/WasmEdge/blob/master/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `master`
- Language: C++ (runtime) with Go / Rust / Node / Python / C SDKs

## Install

```bash
# One-line installer (Linux / macOS, picks the right tarball + plugins)
curl -sSf https://raw.githubusercontent.com/WasmEdge/WasmEdge/master/utils/install_v2.sh | bash

# Homebrew
brew install wasmedge

# Run a .wasm module (interpreter)
wasmedge hello.wasm

# AOT-compile to a native shared object for ~10× cold start
wasmedge compile hello.wasm hello.so
wasmedge hello.so

# Pass arguments + grant directory access (capability-scoped, not blanket)
wasmedge --dir .:. app.wasm input.json

# Run a llama.cpp-shape GGUF model via the WASI-NN GGML plugin
wasmedge --dir .:. \
  --nn-preload default:GGML:AUTO:llama-3.2-3b.gguf \
  llama-chat.wasm

# Use as the OCI runtime for a Wasm container under containerd / crun
sudo crun run --runtime wasmedge mywasmctr
```

## Niche

The "**production-grade Wasm runtime that schedules like a
container**" slot.

`wasmtime` and `wasmer` are the obvious peers; all three implement
the Wasm + WASI spec and can run the same `.wasm` modules. The
differentiation is in the *embedding shape* and *ecosystem
integration*. WasmEdge is the runtime that the CNCF graduated
(2024) specifically because the cloud-native stack picked it as
the default Wasm engine for `containerd` (`runwasi`), `crun`
(`--runtime wasmedge`), Kubernetes (`RuntimeClass: wasmedge`),
KubeEdge, and SuperEdge — so a Wasm pod scheduled by `kubectl
apply` on a mixed cluster lands on a node, gets a
`RuntimeClass: wasmedge`-shaped slot, and the kubelet hands the
OCI bundle to WasmEdge instead of `runc`. That integration is the
load-bearing reason it exists separately from `wasmtime`.

The second axis is the **WASI-NN inference plugin**: the same
`wasmedge` binary that runs your `hello.wasm` will also run a
GGUF llama-shaped LLM via `--nn-preload default:GGML:AUTO:<model>`,
no separate inference server, no Python, no CUDA-toolkit
container — the model is a host-function call, the Wasm module
is your prompt-handling logic. This collapses the "small Wasm
function plus a separate model server" two-process pattern into
a single Wasm runtime call, which is the right shape for
edge / IoT / serverless inference where you do not want a
co-located Python process.

Useful for:

- **Kubernetes Wasm workloads** — `RuntimeClass: wasmedge` plus
  a Wasm-built OCI image lets a small pod start in ~1 ms and use
  ~1 MB of RAM where a Linux-container equivalent costs ~50 MB
  and ~50 ms; appropriate for high-fan-out request handlers.
- **Edge inference** — GGUF / ONNX / PyTorch / TF-Lite via the
  WASI-NN plugin, behind one capability-scoped sandbox, on a
  Raspberry Pi / Jetson / edge gateway where a full Python +
  PyTorch install is too heavy.
- **Serverless platforms** — Vercel / Fastly / Cloudflare-shape
  isolates: WasmEdge cold-starts in microseconds and the OCI
  integration means the same artifact runs on the platform and
  on a `kubectl run` for local debug.
- **Embedding inside a Go / Rust host** — the SDKs let an
  application load untrusted user code as `.wasm`, expose only
  the host functions you choose, and run it with predictable
  memory + CPU bounds (the right shape for "user-supplied
  scripting in a multi-tenant SaaS" without giving up the
  process boundary).

## Why it matters

- **CNCF graduated (2024)** — the Wasm runtime the cloud-native
  stack picked; the `containerd` `runwasi` shim, `crun
  --runtime wasmedge`, Kubernetes `RuntimeClass: wasmedge`,
  KubeEdge, and SuperEdge are first-class integrations not
  community patches.
- **Plugin-driven host functions** — WASI-NN (GGML / ONNX /
  PyTorch / TF-Lite), WASI-Crypto, WASI-HTTP, WasmEdge-RDBMS,
  TensorFlow extensions; capability-scoped at load time
  (`--dir`, `--env`, `--nn-preload`) instead of a blanket
  `--allow-all`.
- **AOT compiler** — `wasmedge compile foo.wasm foo.so` emits
  a native shared object via LLVM; cold start drops by an order
  of magnitude vs the interpreter, the same source `.wasm` is
  the artifact you ship.
- **Embeddable** — Go, Rust, Node, Python, C, C++, Java SDKs;
  the host process registers host functions, the guest Wasm
  calls them, the runtime enforces the capability boundary.
- **OCI-image distribution** — Wasm modules ship as OCI
  artifacts (push / pull with `crane` / `oras` / `docker
  push`), so the same registry workflow that handles container
  images handles Wasm modules; no separate distribution
  channel.
- **Active in 2026** — `0.16.2` (2026-04-15) is the most
  recent release at snapshot time; the project ships every
  6–10 weeks and the WASI-NN plugin set has been the
  fastest-moving surface (GGML backend tracks `llama.cpp`
  closely).
- **Honest scope** — Wasm runtimes are not a drop-in
  replacement for Linux containers; the WASI surface is still
  smaller than POSIX, threads + sockets are
  proposal-and-plugin shaped not POSIX-stable, and a
  CGO-heavy / glibc-heavy program will not compile to Wasm
  without rework. Pick WasmEdge for *new* workloads where the
  Wasm shape is a fit (small request handlers, plugin
  systems, edge inference, multi-tenant scripting), not as a
  "rebuild my legacy container as Wasm" path.
- **vs `wasmtime`** — `wasmtime` (Bytecode Alliance, also
  Apache-2.0) is the spec-reference runtime; pick `wasmtime`
  when you want the canonical implementation and Rust-host
  embedding, pick `wasmedge` when the cloud-native /
  Kubernetes / WASI-NN inference path is what you need.
- **vs `wasmer`** — `wasmer` ships a more end-user-CLI shape
  (`wasmer run`, package manager `wasmer.io`); pick `wasmer`
  when the workflow is "install a Wasm package and run it",
  pick `wasmedge` when the workflow is "embed a runtime in my
  cloud-native stack".
- **Apache-2.0** — permissive; embedding the runtime in a
  commercial product is fine with attribution; the WASI-NN
  GGML plugin links `llama.cpp` (MIT) which has the same
  shape.
