# livekit-agents

> Snapshot date: 2026-04-26. Upstream:
> <https://github.com/livekit/agents>

"**A framework for building realtime voice AI agents.**"
LiveKit Agents is the agent SDK on top of LiveKit's open-source
WebRTC SFU: it wires a realtime audio/video session to an LLM
plus STT, TTS, VAD, turn detection, and tool calls so a few dozen
lines of Python ship a low-latency voice agent that joins a
LiveKit room, listens, thinks, speaks, and can interrupt itself
mid-utterance. Same SDK targets the OpenAI Realtime API or
Gemini Live (single bidirectional WebSocket) and the classic
STT→LLM→TTS pipeline (mix-and-match providers).

## 1. Install footprint

- SDK: `pip install "livekit-agents[openai,silero,deepgram,
  cartesia,turn-detector]~=1.0"` — extras pull plugin packages
  per provider.
- CLI: `python my_agent.py dev` runs the agent worker against a
  LiveKit Cloud or self-hosted SFU; `download-files` pre-fetches
  Silero VAD / turn-detector model weights.
- Self-host: pair with the OSS `livekit-server` (Apache-2.0)
  for fully on-prem voice.

## 2. Repo + version + license

- Repo: <https://github.com/livekit/agents>
- Latest release: **livekit-agents@1.5.6**
- License: **Apache-2.0** —
  <https://github.com/livekit/agents/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (Node SDK in a sibling repo)
- Stars: ~10.2k

## 3. Models supported

LLM: OpenAI (incl. Realtime), Anthropic, Google Gemini (incl.
Live), Groq, Cerebras, Together, AWS Bedrock, Ollama, plus any
OpenAI-compatible endpoint. STT: Deepgram, AssemblyAI, Azure,
Google, OpenAI Whisper, Speechmatics, Gladia, Fal. TTS:
Cartesia, ElevenLabs, OpenAI, Google, PlayHT, Rime, Resemble.
VAD: Silero. Turn detection: in-house transformer.

## 4. Notable angle

**The de-facto OSS framework for voice-first agents with end-to-end
realtime semantics.** Where `pipecat` targets the same problem
with a similar pipeline DSL, livekit-agents leans on LiveKit's
production WebRTC stack (sub-200ms global edge, NACK/PLI, SVC) and
treats interruption / barge-in / end-of-turn as first-class —
the `MultimodalAgent` wrapper around OpenAI Realtime / Gemini
Live collapses the whole STT→LLM→TTS chain into one
bidirectional stream, while the `VoicePipelineAgent` keeps the
classical pipeline mix-and-match for cost / latency / quality
tuning per provider.

## 5. Last verified

2026-04-26 via `gh api repos/livekit/agents`.
