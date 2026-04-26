# agent-cli-local (basnijholt/agent-cli)

- **Repo:** https://github.com/basnijholt/agent-cli
- **Version:** v0.90.0 (2026-04-24)
- **License:** MIT (`LICENSE`)
- **Language:** Python
- **Install:** `uv tool install "agent-cli" -p 3.13` (or `pip install agent-cli`)

> Cataloged here under the slug `agent-cli-local` to avoid colliding with the
> generic `agent-cli` name; the upstream package is `agent-cli`, invoked as
> `agent-cli`, `agent`, or `ag`.

## One-line summary

A suite of **local-first, voice-and-text** AI command-line tools — autocorrect,
transcribe, TTS, voice-edit, RAG proxy, parallel git-worktree dev — that run
entirely on your machine by default.

## What it does

Most CLIs in this catalog are single-purpose coding agents. `agent-cli` is
the opposite: a **toolbox of small commands** sharing a local LLM/ASR/TTS
backend, hotkey-friendly, with optional cloud fallback.

Headline commands:

- `autocorrect` — clean up grammar/spelling on any text.
- `transcribe` / `transcribe-live` — voice → clipboard via Whisper
  (MLX on Apple Silicon, Faster Whisper on Linux/CUDA).
- `speak` — text → speech via Kokoro (GPU) or Piper (CPU).
- `voice-edit` — speak instructions to mutate clipboard text.
- `assistant` — wake-word voice loop.
- `chat` — conversational loop with tool-calling.
- `memory` (+ `memory proxy`) — long-term memory layer in front of any chat.
- `rag-proxy` — chat with your own documents through a local proxy.
- `dev` — parallel coding agents across git worktrees.
- `server` — local ASR/TTS server with both Wyoming and OpenAI-compatible
  protocols, TTL-based memory eviction, multi-platform acceleration.

Audio/data never leaves the machine unless you opt into OpenAI/Gemini.

## When to choose it

- You want a **local-first voice frontend** for LLMs and don't want to send
  audio to a cloud transcription API.
- You bind tools to **system-wide hotkeys** and want a Unix-y suite of
  composable commands rather than one monolith.
- You're on Apple Silicon and want MLX-accelerated Whisper out of the box.
- You want a **single project that bundles ASR + TTS + chat + RAG + memory**
  instead of stitching five tools together.

## When NOT to choose it

- You want a *coding* agent that edits files in a repo — `agent-cli`'s `dev`
  command is the closest, but the project's center of gravity is voice/text
  productivity, not repo work.
- You want zero local setup — running Whisper/Kokoro/Piper locally requires
  models, GPU/MLX runtime, and a non-trivial Python 3.13 environment.
- You want a TUI experience — these are scriptable CLIs designed for
  hotkeys and pipes, not interactive sessions.
