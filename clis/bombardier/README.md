# bombardier

Fast cross-platform HTTP benchmarking tool written in Go.

- **Repo**: https://github.com/codesenberg/bombardier
- **Version**: v1.2.6
- **License**: MIT — `LICENSE`
- **Language**: Go
- **Category**: networking / load-testing / benchmarking

## Why it's interesting

Single-binary HTTP load generator with a friendly TUI-style live progress
display, latency histograms, and percentile breakdowns. Much lighter than
JMeter/Gatling for quick "is this endpoint fast enough?" checks, and easier
to script than `wrk` because all knobs are CLI flags rather than Lua.
Useful when stress-testing local LLM inference servers (vLLM, llama.cpp
server, ollama) before deciding on a deployment topology.

## Install

```sh
go install github.com/codesenberg/bombardier@latest
# or
brew install bombardier
```

## Example invocation

```sh
bombardier -c 50 -d 30s -l http://localhost:11434/api/tags
```
