# llamafile

> Snapshot date: 2026-04. Upstream: <https://github.com/mozilla-ai/llamafile>
> Latest release at snapshot: **v0.10.0** (2026-03-19).
> License file: [`LICENSE`](https://github.com/mozilla-ai/llamafile/blob/main/LICENSE) — Apache-2.0
> with a project-specific exception clause for the bundled `llama.cpp` and
> Cosmopolitan Libc components.

A weights-and-runtime *single executable*. `llamafile` packages a model,
the inference engine (`llama.cpp`), and a tiny web/HTTP/CLI server into
**one Cosmopolitan Libc binary** that runs unchanged on macOS, Linux,
Windows, FreeBSD, OpenBSD, and NetBSD on both x86-64 and arm64 — no
container, no install, no Python, no `pip`. Double-click or `chmod +x &&
./Llama-3.2-3B-Instruct.Q6_K.llamafile` and you have a local OpenAI-
compatible HTTP endpoint on `:8080` plus a chat REPL on stdin.

This entry exists because the rest of the catalog assumes "you already
have an LLM somewhere" — `llamafile` collapses *runtime + weights +
shipping* into the smallest possible artifact. It is what you `scp` to
an air-gapped machine when the answer to "is there an LLM on this box?"
needs to become "yes" in one command.

## 1. Install footprint

- One file. Download a prebuilt `.llamafile` from the releases page
  (model + runtime fused, typically 1–7 GB depending on the quant) or
  build your own with `llamafile-convert <gguf>`.
- Cosmopolitan Libc αcτµαlly portable executable: same byte-for-byte
  binary boots on macOS, Linux, Windows, *BSD, x86-64 and arm64. On
  macOS arm64 you may need the one-line `--rename` workaround documented
  in the upstream README.
- No daemon, no system install, no per-user config. State is whatever
  the embedded `llama.cpp` server writes to its working directory.
- Optional GPU acceleration: NVIDIA CUDA, AMD ROCm, Apple Metal — auto-
  detected at first run, falls back cleanly to CPU.

## 2. License

Apache-2.0 (see [`LICENSE`](https://github.com/mozilla-ai/llamafile/blob/main/LICENSE)).
The bundled `llama.cpp` is MIT and Cosmopolitan Libc is ISC; both notices
ship inside the binary. Individual `.llamafile` distributions inherit
whatever license the embedded model weights carry (Llama-3 community
license, Gemma terms, Apache-2.0 for Mistral, etc.) — read the model
card before redistributing.

## 3. Models supported

Any GGUF model. Prebuilt `.llamafile`s are published for Llama 3.x,
Gemma 2/3, Mistral, Mixtral, Phi-3.x, Qwen 2.5 / 2.5-Coder, DeepSeek
variants, TinyLlama, and the LLaVA vision family. Bring-your-own:
`llamafile-convert path/to/model.gguf` produces a fresh executable.

## 4. MCP support

**No.** `llamafile` is the runtime layer; MCP belongs to the agent
client. Point any MCP-aware CLI in this catalog (`opencode`, `crush`,
`goose`, `claude-code`, `gemini-cli`) at the local `:8080/v1` endpoint
and let it handle the protocol.

## 5. Sub-agent model

None — runtime only.

## 6. Telemetry stance

**Off. No phone-home at all.** The binary does not ship analytics. The
only outbound network connection is whatever you point an HTTP client
at; the server itself listens on `127.0.0.1:8080` by default.

## 7. Prompt-cache strategy

Inherits `llama.cpp`'s in-process KV cache (`--ctx-size`, `--n-keep`,
slot-based context shifting in server mode). No provider-side prefix
caching — there is no provider, only your local CPU/GPU.

## 8. Hot keybinds

There is no agent TUI. Three surfaces:

| Surface | Invocation | Purpose |
|---------|------------|---------|
| Web UI | `./model.llamafile` then open `http://127.0.0.1:8080` | Browser chat |
| HTTP API | `POST :8080/v1/chat/completions` | OpenAI-compatible endpoint |
| CLI mode | `./model.llamafile --cli -p "..."` | One-shot stdout |

Standard `llama.cpp` server flags apply (`-c` for context, `-ngl` for
GPU layer offload, `--temp`, `--repeat-penalty`, `--mlock`, etc.).

## 9. Killer feature, weakness, when to choose

- **Killer:** **one file, every OS, every CPU/GPU, weights included.**
  No other entry in the catalog ships *runtime + weights + multi-OS
  bootability* in a single artifact. `ollama` needs a daemon; `ramalama`
  needs Podman/Docker; `LocalAI` needs a config tree. `llamafile`
  needs `chmod +x`.
- **Weakness:** **giant artifacts, slow to update.** A Q6_K Llama-3.1-8B
  llamafile is ~6 GB; you re-download the whole executable to upgrade
  the runtime even if the weights did not change. `ollama pull` only
  fetches the layers that changed — `llamafile` has no such layering.
- **Choose it when:** you need to give a non-technical user, an air-
  gapped machine, or a fresh laptop a working local LLM in one file
  with zero install steps; or you want a single artifact that boots on
  every desktop OS your team uses without per-platform packaging.

## Pitfalls observed in real use

1. **macOS arm64 needs the `--rename` workaround for files >4 GB** —
   Apple's exec loader rejects the fat Cosmopolitan format for large
   binaries; the upstream README documents a one-line fix using
   `cosmocc`'s `assimilate` tool.
2. **Windows Defender flags first launch.** Cosmopolitan binaries trip
   heuristic AV scans because the same bytes execute as ELF, Mach-O,
   and PE. Add an exclusion or run from a developer terminal.
3. **GPU offload defaults to off.** You must pass `-ngl 999` (or the
   appropriate layer count) to push work onto Metal/CUDA/ROCm; without
   it you get CPU-only inference even on a beefy GPU.
4. **The bundled web UI is `llama.cpp`'s, not a polished product.** For
   a real chat experience point `oterm`, `elia`, `aichat`, or any other
   catalog client at `http://127.0.0.1:8080/v1`.
5. **`.llamafile` files are not signed.** Verify SHA-256 against the
   upstream releases page before running anything you did not build
   yourself; the executable runs with full user privileges on first
   launch.
