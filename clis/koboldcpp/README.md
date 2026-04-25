# koboldcpp

> Snapshot date: 2026-04. Upstream: <https://github.com/LostRuins/koboldcpp>
> Last verified release: **v1.112.2** (2026-04-20, tag SHA `b31877e8ec8a`).
> License file: [`LICENSE.md`](https://github.com/LostRuins/koboldcpp/blob/concedo/LICENSE.md) — AGPL-3.0.

A single-file C++ **local LLM runtime** built on `llama.cpp`. Where
`ollama` optimises for "daemon + library + opinionated registry",
`koboldcpp` optimises for "one binary you can email someone" — it
ships as a portable executable that bundles a llama.cpp inference
engine plus a built-in HTTP server with both KoboldAI-native and
OpenAI-compatible APIs at `http://localhost:5001/v1`.

It earns its catalog slot for two practical reasons: (1) it is the
fastest way to expose a downloaded `.gguf` file as an
OpenAI-compatible endpoint without installing Python or a daemon, and
(2) it ships first-class support for sampler knobs (`min-p`,
`top-a`, dynamic temperature, DRY, mirostat) that the other local
runtimes hide.

## 1. Install

```bash
# macOS / Linux: prebuilt binary
curl -L https://github.com/LostRuins/koboldcpp/releases/latest/download/koboldcpp-mac-arm64 \
  -o koboldcpp && chmod +x koboldcpp
# Windows: download koboldcpp.exe from the releases page.
# Or build from source:
git clone https://github.com/LostRuins/koboldcpp.git && cd koboldcpp && make
```

No package manager entry, no install script — the binary is the
install.

## 2. Example

```bash
# Serve any GGUF as an OpenAI-compatible endpoint:
./koboldcpp --model qwen2.5-coder-7b-q4_k_m.gguf --port 5001 --gpulayers 99

# Then point any catalog CLI at it:
OPENAI_API_BASE=http://localhost:5001/v1 OPENAI_API_KEY=dummy \
  llm -m gpt-3.5-turbo "summarize this file" < notes.md
```

## 3. Honest limitation

AGPL-3.0 makes embedding `koboldcpp` inside a hosted product
legally awkward — if you serve its HTTP endpoint to third parties
over a network you owe them the source of any modified
`koboldcpp`. For private / single-user / CLI-pipeline use this is a
non-issue, but it rules out shipping `koboldcpp` as the runtime
inside a commercial SaaS without a copyleft-clean rewrite. Pick
`llamafile` or `ollama` (Apache-2.0 / MIT) if that matters.
