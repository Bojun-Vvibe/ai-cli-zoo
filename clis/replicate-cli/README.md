# replicate-cli

> Snapshot date: 2026-04. Upstream: <https://github.com/replicate/cli>
> Binary name: `replicate`

`replicate-cli` is the **hosted-inference-as-a-shell-command** entry of
the catalog. Replicate the platform exposes thousands of community and
first-party models (image, video, audio, LLMs, embedding) behind a
uniform `POST /v1/predictions` API; `replicate` the binary turns that
API into Unix-pipe ergonomics: `replicate run owner/model -i
prompt="a cat"` reads inputs as flags, streams progress to stderr, and
prints the prediction output to stdout. That single shape covers
text-out, file-out (the binary downloads URLs to disk), and
streaming-token-out models without per-model ceremony.

The CLI is a single static Go binary, which puts it in the same
distribution class as [`tgpt`](../tgpt/), [`mods`](../mods/), and
[`ollama`](../ollama/) — install with `brew`, set one env var, done.
Unlike the bring-your-own-provider CLIs, the model surface here is
**Replicate's catalog specifically**: if the model you want is not on
Replicate, this tool does nothing for you; if it is, this is the
shortest path from "I read about a model on Twitter" to "it ran on my
input and the result is on my disk".

## 1. License

Apache-2.0. License file: [`LICENSE`](https://github.com/replicate/cli/blob/main/LICENSE).

## 2. Latest version

`v0.8.0` (released 2024-06-20). Release cadence is slow — the binary
is a thin wrapper over a stable HTTP API, so most platform-side
improvements land without needing a new CLI release. Track `main` if
you want bleeding-edge subcommands.

## 3. Install

```bash
brew install replicate                             # macOS / Linuxbrew
# or download from https://github.com/replicate/cli/releases

export REPLICATE_API_TOKEN=r8_...                  # from https://replicate.com/account/api-tokens
replicate run stability-ai/sdxl \
  -i prompt="a corgi astronaut, photorealistic" \
  -i width=1024 -i height=1024
```

`replicate run` resolves the latest version of `owner/model` by
default; pin a specific version with `owner/model:abc123...` to make a
script reproducible. Output files are written into the current
directory with names like `output.png`, `output.0.png`, etc.

## 4. When to choose

- You want to **try a model that is on Replicate** with one shell line
  and no Python/JS scaffolding around the SDK.
- You want **hosted GPU inference billed by the second**, not by the
  hour — Replicate's cold-boot/per-second pricing fits intermittent
  CLI use better than renting a GPU box.
- You are stitching image / audio / video models into a **shell
  pipeline** (`replicate run ... | xargs -I{} ...`) where each step
  produces a file the next step consumes.
- Avoid if you are doing **pure-text LLM chat** — `mods`, `llm`,
  `tgpt`, `aichat`, or any provider-native CLI is a better fit. Use
  `replicate-cli` when the model output is a *file* (image, audio,
  video) or when the model is Replicate-exclusive.
- Avoid if you need **on-prem / no-egress** inference — everything
  runs on Replicate's GPUs, your inputs leave your network.
