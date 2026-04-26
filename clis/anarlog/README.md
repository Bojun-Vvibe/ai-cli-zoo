# anarlog (formerly hyprnote)

> Snapshot date: 2026-04. Upstream: <https://github.com/fastrepl/anarlog>

A **local-first AI meeting notetaker** that captures system + microphone
audio during a call, transcribes locally with a bundled Whisper-family
model, and runs an LLM summarization pass over the transcript — all
without sending audio off the device by default. Originally released as
**Hyprnote**; renamed to **Anarlog** in early 2026 (the binary, repo,
and Tauri app id all moved). Ships as a desktop app (Tauri) with a
companion CLI for headless transcription / summarization runs.

## 1. Install footprint

- macOS / Windows / Linux Tauri desktop app from
  <https://github.com/fastrepl/anarlog/releases> — `Anarlog_<version>.dmg`
  / `.msi` / `.AppImage`. ~120 MB installed.
- Bundled local STT (Whisper-family GGML), bundled local LLM runtime
  (llama.cpp under the hood), optional remote provider keys for users
  who want a frontier model on the summarization step.
- CLI (`anarlog --help`) drops out of the desktop app bundle on macOS
  (`/Applications/Anarlog.app/Contents/MacOS/anarlog`); on Linux the
  AppImage exposes the same binary.
- First launch downloads the chosen Whisper weights into
  `~/Library/Application Support/com.fastrepl.anarlog/models/`
  (or the XDG equivalent) — explicit consent prompt, no silent fetch.

## 2. Repo, version, license

- Repo: <https://github.com/fastrepl/anarlog>
- Version checked: **`desktop_v1.0.24`** (released 2026-04-16)
- License: **MIT**. License file:
  <https://github.com/fastrepl/anarlog/blob/main/LICENSE>
- Default branch: `main`
- The repo is a Tauri monorepo: `apps/desktop/` (Rust + TS), `crates/`
  (Rust core: audio capture, transcription, summarization), `plugins/`
  (extension surface).

## 3. Models supported

- **STT**: bundled Whisper-family GGML weights (tiny / base / small /
  medium / large-v3) via whisper.cpp; user picks at first run, can
  swap in a custom GGML file.
- **LLM (summarization / chat over transcript)**: local llama.cpp
  models (any GGUF), or optional remote providers (OpenAI, Anthropic,
  Gemini, OpenRouter, any OpenAI-compatible base URL).
- Default install is fully offline; remote providers are opt-in,
  per-template, and the UI shows which model produced each summary.

## 4. MCP support

Not first-party as of this snapshot. The plugin surface is a Tauri
extension API (Rust / TS); a community plugin exposing transcripts as
an MCP resource exists but is not bundled.

## 5. Sub-agent model

None. Anarlog is a vertical product (capture → transcribe → summarize),
not an agent framework. Templates can chain multiple LLM passes
(summary → action items → follow-up email draft) but each is a
straight prompt, not a tool-use loop.

## 6. Telemetry stance

Off. Audio capture, transcription, embeddings, and the local LLM all
run on-device. Crash reports and product analytics are explicit opt-in
during onboarding; the OSS build with `--features no-telemetry`
strips the network paths entirely. Remote LLM providers, when enabled,
see only the *transcript text* you choose to send (audio never leaves
the device under any configuration).

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Genuinely local meeting capture with a clean
desktop UX.** Most "AI notetaker" tools either (a) require a SaaS
backend that ingests your audio, (b) are a browser extension that
breaks on Zoom updates, or (c) are a hand-rolled `whisper.cpp` script
in a terminal. Anarlog is none of those — it captures system audio
*and* mic on macOS / Windows / Linux, runs Whisper locally, runs the
summarizer locally if you want, and ships as a polished MIT-licensed
desktop app with a CLI escape hatch.

**Weakness.** The transcription / summarization quality ceiling is
the local model you pick — Whisper-large + a 70B local LLM is great
on a Mac with enough RAM and slow on anything else. The 2026
Hyprnote → Anarlog rename means older blog posts, package indexes,
and cached search results still point at the old name; the previous
binary is deprecated, not maintained. System-audio capture on Linux
relies on PipeWire / PulseAudio loopback configuration that the app
can't fully automate. No multi-device sync (by design — there is no
backend to sync to).

**When to choose.**
- You sit in many meetings, you want a transcript + summary on your
  own disk, and you specifically do **not** want a SaaS notetaker
  joining the call as a bot.
- You need a **local** transcription pipeline with a real GUI (your
  team won't run a `ffmpeg | whisper-cpp` command line).
- You want an MIT-licensed base you can fork for vertical workflows
  (legal, medical, journalism) where audio off-device is a hard no.

**When to skip.**
- You want a SaaS notetaker that joins meetings as a bot and writes
  to a shared workspace — Anarlog deliberately doesn't do that.
- You only need raw transcription, no UI →
  [`whisper.cpp`](https://github.com/ggerganov/whisper.cpp) directly.
- You want an LLM-driven *agent* over your meetings (search, query,
  cross-reference) — pair Anarlog with [`khoj`](../khoj/) /
  [`paper-qa`](../paper-qa/) over the exported transcripts.

## 8. Compared to neighbors

| Tool | Shape | Audio capture | Local-first |
|------|-------|---------------|-------------|
| anarlog | Tauri desktop app + CLI | System + mic | Yes (audio never leaves device) |
| [khoj](../khoj/) | Personal AI over docs | None — operates on text | Yes (self-hosted) |
| [paper-qa](../paper-qa/) | RAG CLI for documents | None | Yes (local) |
