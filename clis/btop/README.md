# btop

> **Resource monitor TUI that draws a single dense braille-character
> dashboard for CPU (per-core), memory, swap, disks, network, GPU
> (NVIDIA / AMD / Intel), and a sortable / filterable / tree-view
> process list — `htop` plus `nvtop` plus `iftop` plus `iotop` in one
> mouse-clickable, theme-able C++ binary.** Pinned to **v1.4.6**,
> Apache-2.0
> ([LICENSE](https://github.com/aristocratos/btop/blob/main/LICENSE)).

- **Repo:** https://github.com/aristocratos/btop
- **Latest version:** v1.4.6
- **License:** Apache-2.0 (`LICENSE` at repo root, SPDX `Apache-2.0`)
- **Category:** `system-monitor` / `tui`
- **Language:** C++
- **Platforms:** Linux, macOS, FreeBSD, OpenBSD, NetBSD (Windows via WSL)

## What it does

`btop` is a full-screen TUI resource monitor whose layout is four
resizable, individually-toggleable boxes — CPU, memory + disks,
network, processes — drawn with braille / quadrant Unicode for
sub-character resolution graphs. The CPU box plots per-core usage
plus aggregate frequency, temperature (per-core when the kernel
exposes it), and uptime; the memory box stacks RAM + swap bars
beside per-disk I/O graphs and free-space gauges; the network box
plots up / down on independent autoscaling axes per interface; the
process box is `htop`-style but adds tree mode (`e`), sortable
columns, click-to-sort headers, click-to-kill, fuzzy filter (`f`),
and per-process detail panel (`Enter`) with full command line, user,
parent, threads, open-files count, and a dedicated CPU+mem
historical graph for the selected PID. GPU support (1.3+) draws a
fifth box per device with utilisation, VRAM, temperature, and power
draw. Themes are plain text files; `+` / `-` keys live-cycle the
bundled set, and a config-file flag picks the default. Mouse,
keyboard, and `vim`-style navigation all work; SIGWINCH reflows the
layout cleanly.

## Why included

The catalog already carries [bottom](clis/bottom/) (Rust, lighter,
MIT) for the "I want a quick `htop` upgrade" slot, but `btop` owns
the "single pane of glass while an LLM agent is hammering my
laptop" slot — running `claude` or `aider` against a 100k-line
codebase pegs CPU, fills RAM with a vector store, drives disk I/O
through ripgrep, and saturates the network on tool-call retries,
all at once. `btop`'s four-box layout shows every one of those
axes simultaneously without flipping windows, and the per-process
detail view answers "which child of which agent is the one
chewing 9 GB" in two keystrokes. GPU support matters for the
local-model half of the catalog (Ollama, llama.cpp, vLLM) where
"is the model actually on the GPU and how much VRAM is left for
context" is a question `nvidia-smi` answers badly and `btop`
answers in the same dashboard as everything else.
