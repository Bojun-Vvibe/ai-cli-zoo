# daktilo

> Snapshot date: 2026-05. Upstream: <https://github.com/orhun/daktilo>

**Turn your keyboard into a typewriter.** Daktilo (Turkish for
"typewriter") is a small Rust daemon that listens for global key
events and plays a sound sample per keystroke — clacks for the
letters, a bell `ding` at the end of a line, a longer return-carriage
slide on Enter. It ships with multiple configurable presets
(IBM Model M, Olympia SM3, mechanical Cherry MX Blue, plus a "boxy
typewriter" sample pack) and a TOML config so you can map your own
samples per key range.

## Repo + version + license

- Repo: <https://github.com/orhun/daktilo>
- Latest release: **`v0.6.0`** (2024-04-10)
- License: **Apache-2.0** (dual-licensed Apache-2.0 / MIT) —
  <https://github.com/orhun/daktilo/blob/main/LICENSE-APACHE>
- License paths in repo: `LICENSE-APACHE`, `LICENSE-MIT`
- Default branch: `main`
- Language: Rust

## Install

```bash
# cargo
cargo install daktilo

# Run with a built-in preset
daktilo --preset typewriter

# List bundled presets and define your own in ~/.config/daktilo/daktilo.toml
daktilo --list-presets
daktilo --preset boxy-typewriter --device-name "AT Translated Set 2 keyboard"
```

## Niche

The "**audio feedback for your keyboard, configured per-app, with
per-key sample mapping**" slot. Where [`bucklespring`](https://github.com/zevv/bucklespring)
plays a single buckling-spring sample and stops there, daktilo lets
you bind different sample packs to different key ranges (alpha vs.
modifier vs. Enter), define multiple `[[sound_preset]]` blocks and
hot-swap between them, and filter by input device so a USB keyboard
sounds different from your laptop's built-in one. It's also the rare
entry in this niche that's actively maintained as of 2024 and ships
prebuilt binaries for Linux + macOS + Windows.

## Why it matters

- **Multi-preset config in one TOML** — `[[sound_preset]]` blocks
  with `name`, `key_config` (regex over key names), `files` (sample
  pool with weights), and `enabled_for` / `disabled_for` window-
  title regex so the typewriter clack only plays in your editor and
  not when Slack is focused.
- **Per-key sample distribution** — every keypress picks a sample
  from a weighted pool, so the audio doesn't loop noticeably during
  a long writing session.
- **Variations beyond typewriter** — bundled presets cover IBM
  Model M, Cherry MX Blue, and a "drum kit" mode that maps keys to
  percussion samples (useful for live coding / pair programming
  visibility).
- **Quality-of-life author** — same maintainer as
  [`git-cliff`](../git-cliff/), [`menyoki`](../menyoki/),
  [`kmon`](../kmon/), and [`binsider`](../binsider/); the project
  has the same single-binary, well-documented, prebuilt-release
  shape as the rest of orhun's catalog.
- **No telemetry, no network** — pure local audio playback; the
  binary doesn't open any sockets.
