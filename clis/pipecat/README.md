# pipecat

> Snapshot date: 2026-04. Upstream: <https://github.com/pipecat-ai/pipecat>

An open-source Python framework for **real-time, voice-first
multimodal conversational AI agents** — the orchestration
substrate for "phone bot you can have a natural duplex
conversation with", "in-game NPC that listens and replies
sub-second", or "video-call avatar driven by an LLM". Pipecat
wires speech-to-text, an LLM, text-to-speech, vision, and
transport (WebRTC / WebSocket / Twilio / Daily / LiveKit) into a
single asynchronous **Pipeline** of `FrameProcessor` nodes that
push typed `Frame` objects (audio, text, transcription, function
call, image, control) at each other.

It is the catalog's reference for **production voice-agent
orchestration where end-to-end latency, interruptions, and
turn-taking are first-class concerns** — not a coding agent.

## 1. Install footprint

- `pip install pipecat-ai` (Python 3.10+).
- Modular extras: `pipecat-ai[daily,openai,deepgram,cartesia,silero]`
  pulls only the transports / providers / VAD you need; ~150 MB
  with a typical voice stack, more if you add local Whisper or
  RVC.
- Models: OpenAI / Anthropic / Gemini / Groq / Together /
  OpenRouter / Ollama / vLLM for the LLM hop; Deepgram /
  AssemblyAI / Whisper (cloud or local) / Azure / Google /
  Gladia for STT; Cartesia / ElevenLabs / Rime / PlayHT / OpenAI
  TTS / Azure / Google / Piper for TTS; Silero / WebRTC VAD /
  smart-turn for voice activity + turn detection.
- Transports: Daily, LiveKit, Twilio (telephony), FastAPI
  WebSocket, local audio.

## 2. Repo, version, license

- Repo: <https://github.com/pipecat-ai/pipecat>
- Version checked: **v1.0.0** (released 2025-Q4, the first
  stable major after a long 0.x stabilization window).
- HEAD pinned at this snapshot:
  `6266c026a6da987ec88930a1f066e9b23a607617`.
- License: BSD-2-Clause. License file at
  [`LICENSE`](https://github.com/pipecat-ai/pipecat/blob/main/LICENSE).

## 3. What it actually does

Pipecat models a real-time agent as a directional pipeline of
`FrameProcessor` nodes that pass `Frame` objects asynchronously:

```
Transport.input → VAD → STT → LLM (with tools) → TTS → Transport.output
```

Each node is a small async class with `process_frame(frame,
direction)` that can transform / forward / drop / split frames.
Built-in services wrap each provider (`OpenAILLMService`,
`DeepgramSTTService`, `CartesiaTTSService`, …) so swapping
providers is one line. Function calling is a first-class
`FunctionCallProcessor` that registers tool schemas with the
LLM service and routes results back into the LLM context.

Transports (`DailyTransport`, `LiveKitTransport`,
`TwilioTransport`, `FastAPIWebsocketTransport`) deliver audio
frames from the user side and accept synthesized frames from the
agent side, handling network jitter, packet loss, and RTC
signalling.

Two distinguishing capabilities:

1. **Interruption handling.** When the user starts speaking
   mid-response, an `InterruptionFrame` propagates through the
   pipeline: TTS stops mid-sentence, the partial spoken text is
   logged into the LLM context as "user interrupted at
   word N", and the next user utterance is processed cleanly.
   This is the hard problem voice agents fail on.
2. **Turn-taking with smart-turn.** Beyond simple VAD, an
   optional ML model predicts whether the user has *finished*
   speaking versus *paused mid-thought*, eliminating the
   "agent talks over me at every comma" failure mode.

## 4. MCP support

None first-party as of v1.0. The tool-call surface is
provider-native function calling (OpenAI / Anthropic / Gemini
schemas) wired through `FunctionCallProcessor`. An MCP-client
adapter is on the roadmap but not shipped; users currently bridge
by writing a `FrameProcessor` that proxies to an MCP server.

## 5. Sub-agent model

Single-pipeline by default; the `ParallelPipeline` /
`SyncParallelPipeline` primitives let you fan out frames to
multiple sub-pipelines (e.g. STT + sentiment classifier in
parallel, results merged before the LLM hop). No spawned worker
agents — the model is *one* always-on conversational loop, not a
planner/executor split. Multi-bot orchestration (e.g. caller +
two specialized expert bots) is done at the transport layer by
joining multiple bots into the same Daily / LiveKit room.

## 6. Telemetry stance

Off by default in the OSS library. OpenTelemetry instrumentation
is opt-in (`enable_tracing=True` on the runner) and exports to
any OTel collector — common targets are Langfuse, Phoenix,
Logfire, Honeycomb. Egress = whichever STT / LLM / TTS / transport
providers you configure (audio leaves your process for the cloud
provider unless you wire local Whisper + local LLM + local Piper).

## 7. Token / context strategy

LLM context is managed by the framework's `LLMContext` object,
which accumulates user transcriptions + assistant responses +
function calls/results across turns. No automatic truncation —
the developer supplies a custom `LLMContextAggregator` if a long
call (>30 minutes) needs windowed context or summarization.
Latency-critical: every extra token in context is extra TTFT
on every turn, which the user *hears* as silence. The default
recommendation is short system prompts (<500 tokens) and aggressive
context trimming after N turns.

## 8. Hot keybinds

None — Pipecat is a Python framework, not a TUI. The interactive
surface is the audio call itself; debugging happens via
log streams, OTel spans, and the `pipecat-ai/pipecat-flows`
sibling project's visual editor for declarative dialog flows.

## 9. Killer feature, weakness, when to choose

**Killer feature.** End-to-end **sub-second voice agent
latency** as a designed-in property: pipeline-of-frames model
streams partial STT into partial LLM into partial TTS so the
agent starts speaking before the user has finished the next
word, with **interruption handling and smart turn detection
that actually work** in production telephony. The provider
matrix is the broadest in any voice-agent framework — every
serious STT / LLM / TTS / transport vendor has a first-class
`Service` class so vendor swaps don't require rewriting the
pipeline.

**Weakness.** Voice-first means you write more boilerplate for a
"text chatbot with optional voice" than you would in a
text-first framework like `pydantic-ai` or `langgraph`. The
pipeline-of-frames mental model takes a session to internalize
(everything is async, everything is a frame type, processors
must respect frame direction). Provider sprawl is also
config-sprawl: a typical production setup juggles credentials
for transport + STT + LLM + TTS + VAD across 4-5 vendors.

**Choose pipecat when** the product is a real-time voice or
video agent (phone bot, in-game NPC, video call assistant,
drive-through ordering kiosk, in-car voice UX) and end-to-end
latency + interruption handling + multi-vendor portability are
the load-bearing concerns. **Choose something else when** the
agent is text-first (use [`pydantic-ai`](../pydantic-ai/) /
[`openai-agents-python`](../openai-agents-python/) /
[`langgraph`](../langgraph/)), when LiveKit's own
`livekit-agents` ([already in catalog](../livekit-agents/)) is
sufficient and you don't need provider portability beyond
LiveKit's stack, or when a managed voice-bot platform (Vapi,
Retell, Bland) is acceptable and you want to avoid running the
orchestration yourself.
