# 01

> Snapshot date: 2026-04. Upstream: <https://github.com/openinterpreter/01>

The OpenInterpreter team's voice-first agent. Where `open-interpreter` is a
text REPL that writes-and-runs code on your machine, `01` wraps that same loop
in a push-to-talk audio surface: speak an instruction, the agent transcribes
it, plans, executes shell / Python on the host, and replies through TTS. Runs
on desktop, mobile, or an ESP32 hardware client talking to a host server.

## 1. Install footprint

- `pip install 01OS` (pulls the server). The client targets are separate:
  desktop app, iOS/Android shells, and an ESP32 firmware blob.
- Python 3.10+. macOS, Linux, Windows for the server.
- Local STT (faster-whisper) and TTS (piper / coqui) by default, downloaded on
  first run. Optional cloud STT/TTS (OpenAI, ElevenLabs, Deepgram).
- State in `~/.01/` (config, conversation history, model cache).

## 2. License

AGPL-3.0. License file: [`LICENSE`](https://github.com/openinterpreter/01/blob/main/LICENSE).

## 3. Latest version

`0.01.1` (only published tag in <https://github.com/openinterpreter/01/tags>;
the project versions through git directly between rare tagged drops).

## 4. Models supported

Inherits OpenInterpreter's LiteLLM-routed provider list: OpenAI, Anthropic,
Gemini, local llama.cpp / Ollama / LM Studio, any OpenAI-compatible endpoint.
The model receives transcribed text plus current shell context and emits code
to run.

## 5. MCP support

No. Tools come from the OpenInterpreter executor (shell, Python, file I/O,
optional browser); MCP is not wired into the audio loop.

## 6. Sub-agent model

None — single agent, single voice session, code-execution loop is the same
as OpenInterpreter's.

## 7. Telemetry stance

Off. The local server does not phone home; STT / TTS / LLM egress only
happens when you configure a cloud provider for any of them.

## 8. When to choose it

- You want a hands-free voice surface for the OpenInterpreter agent loop —
  dictate at the laptop, drive a build, walk away.
- You are prototyping ambient / always-listening agent hardware (the ESP32
  client is a real reference design, not a demo).
- You want a self-hostable Siri / Alexa replacement whose "skills" are
  arbitrary Python and shell, not a closed skill marketplace.

## 9. When to skip it

- You want a text agent — go straight to `open-interpreter` and skip the
  audio stack.
- AGPL is a problem for your distribution model. The server is AGPL-3.0;
  embedding it in a commercial product without source release is a license
  conversation.
- You need MCP tool integration — `01` does not ship that surface.
