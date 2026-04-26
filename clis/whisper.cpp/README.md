# whisper.cpp

> Snapshot date: 2026-04. Upstream: <https://github.com/ggml-org/whisper.cpp>

`whisper.cpp` is Georgi Gerganov's C/C++ port of OpenAI's Whisper
speech-to-text model. It is the **runtime substrate** under most
locally-running transcription tools you have ever used — voice memos
on macOS share extensions, Obsidian voice plugins, dictation overlays,
and the audio side of more than a few "AI assistant" desktop apps all
shell out to a `whisper-cli` binary built from this repo.

It is in scope for this catalog because it ships a real,
natural-language-shaped CLI (`whisper-cli`) that takes audio and emits
text/JSON/SRT/VTT — the "speech → prompt" half of every voice-driven
coding workflow.

## 1. Install footprint

- **Build from source** (recommended, ~1 min): `git clone
  https://github.com/ggml-org/whisper.cpp && cd whisper.cpp && cmake
  -B build && cmake --build build -j --config Release`. Produces
  `build/bin/whisper-cli`.
- **Apple Silicon / Metal**: enabled by default on macOS; CoreML
  encoder is opt-in with `-DWHISPER_COREML=1`.
- **NVIDIA**: `cmake -B build -DGGML_CUDA=1`.
- **Models**: `bash ./models/download-ggml-model.sh base.en` (and
  `small`, `medium`, `large-v3`, `large-v3-turbo`, etc.). Models live
  in `models/ggml-*.bin`.
- **Homebrew**: `brew install whisper-cpp` for the pre-built binary
  + manpage; you still need to download a `.bin` model separately.

Zero Python, zero CUDA SDK on macOS, zero network access at inference
time. The binary plus a model file is the entire dependency surface.

## 2. License

MIT (file: [`LICENSE`](https://github.com/ggml-org/whisper.cpp/blob/master/LICENSE)).

## 3. What is in the box

- `whisper-cli` — the headline CLI: audio in, transcript out, with
  flags for language, threads, GPU layers, beam size, word-level
  timestamps (`-ml 1`), translation, diarisation (`--tinydiarize`),
  speaker turns, and output formats (`--output-txt`, `-osrt`, `-ovtt`,
  `-ojson`, `-olrc`, `-ocsv`).
- `whisper-stream` — live microphone streaming (uses SDL2).
- `whisper-bench` — model + hardware benchmarking.
- `whisper-server` — OpenAI `/v1/audio/transcriptions`-compatible HTTP
  endpoint. This is the integration point most other CLIs in this
  catalog use when they want "voice → text → LLM" without bundling
  Whisper themselves.
- A `talk-llama` example that wires whisper.cpp to llama.cpp for a
  full local voice assistant loop.
- GGML quantised model variants from `tiny` (~75 MB) up to
  `large-v3` (~3 GB) with Q4 / Q5 / Q8 quantisations of each.

## 4. One-liner usage

```bash
# Transcribe a 16 kHz mono WAV to plain text using base.en
./build/bin/whisper-cli -m models/ggml-base.en.bin -f samples/jfk.wav

# Live mic dictation, English, 6 threads, into stdout
./build/bin/whisper-stream -m models/ggml-base.en.bin -t 6 --step 500 --length 5000

# Run as an OpenAI-compatible transcription server on :8080
./build/bin/whisper-server -m models/ggml-large-v3-turbo.bin --host 0.0.0.0 --port 8080
```

Most LLM CLIs in this zoo ([`llm`](../llm/), [`mods`](../mods/),
[`aichat`](../aichat/)) can pipe straight from `whisper-cli` output:
`whisper-cli -m models/ggml-base.en.bin -f mic.wav -otxt -of - | llm
"summarise"`.

## 5. Source-of-truth pin

- Verified default branch: `master`
- Latest release tag at snapshot: **`v1.8.4`**
- HEAD SHA at snapshot: `fc674574ca27cac59a15e5b22a09b9d9ad62aafe`
  (committed `2026-04-20T05:12:57Z`)
- License file path: `LICENSE` (SPDX: MIT)
- Verified UTC: `2026-04-26T19:16:33Z` via `gh api repos/ggml-org/whisper.cpp`

## 6. Where it fits in the zoo

`whisper.cpp` is the *speech-input* peer to [`llama.cpp`](../llama.cpp/)
(text generation) and [`mlx-lm`](../mlx-lm/) (Apple-native fast
decode). It does not generate; it transcribes. Pair it with anything
in this catalog that takes stdin and you have a fully-local voice →
LLM → action pipeline with no cloud round-trip.
