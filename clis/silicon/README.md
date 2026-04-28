# silicon

> **Render source code to a PNG that looks like a real editor
> screenshot — without taking a screenshot.** A single Rust
> binary that accepts code on stdin (or a file), syntax-
> highlights it via `syntect`, and emits a PNG with window
> chrome, line numbers, padding, watermark, and a configurable
> background. Pinned to **v0.5.3**
> ([LICENSE](https://github.com/Aloxaf/silicon/blob/master/LICENSE),
> MIT).

Source: <https://github.com/Aloxaf/silicon>

## Category

Documentation / dev-tooling. Sits where a `carbon.now.sh` web
upload, a Cmd-Shift-4 of your terminal, and `pygmentize | wkhtmltoimage`
all overlap — but as a single offline binary that fits in a
script. No browser, no network, no font-rendering surprises.

## Why catalog-worthy

Three concrete dev workflows it solves cleanly:

1. **Bug-report screenshots that aren't screenshots.** Paste a
   stack trace into a script, get a PNG — no font hinting drift
   between your retina display and a colleague's external
   monitor, no accidental capture of a Slack notification, no
   leaking unrelated terminal scrollback.
2. **Reproducible README/social-card art.** Generating the
   "screenshot of code" on the README from CI means it's
   always in sync with the actual snippet, with the same
   theme, padding, and font on every regen. No "the
   screenshot is from v1.3, the code now is v2.1" drift.
3. **Air-gapped / offline.** Unlike `carbon.now.sh`, your
   pre-release source never leaves the machine.

## What it does

Pipe (or pass) source code → silicon picks a syntect syntax
(`-l rust`/`-l py`/auto-detect by filename), renders it with
one of the bundled `bat`-compatible themes, draws a configurable
window frame (macOS-style traffic lights optional), optionally
overlays line numbers / a watermark / a shadow, and writes a
PNG. Output goes to a file (`-o`) or to the system clipboard
(`--to-clipboard`).

## Install

```bash
# Homebrew
brew install silicon

# Cargo (needs harfbuzz + freetype dev libs on Linux)
cargo install silicon --locked

# Verify
silicon --version    # silicon 0.5.3
```

## License

MIT — see
[LICENSE](https://github.com/Aloxaf/silicon/blob/master/LICENSE).
Permissive, no attribution required for binaries or output PNGs.

## One Concrete Example

```bash
# stack trace from clipboard → highlighted PNG back to clipboard
pbpaste | silicon --language python --theme Dracula --to-clipboard

# diff hunk from git → committed-into-repo screenshot, no GUI involved
git diff HEAD~1 -- src/lib.rs \
  | silicon --language diff --theme 'Solarized (dark)' \
            --no-window-controls --pad-horiz 20 --pad-vert 20 \
            -o docs/img/lib-rs-diff.png

# CI-generated social card, deterministic across runs
silicon examples/quickstart.rs \
  -l rust -t OneHalfDark \
  --background '#1e1e2e' --shadow-color '#00000088' --shadow-blur-radius 30 \
  --highlight-lines '12;15-18' \
  -o assets/social/quickstart.png

# rebuild syntax/theme cache after dropping a new .tmTheme into ~/.config/silicon/themes
silicon --build-cache
```

## Niche It Fills

`screencapture` / Cmd-Shift-4 captures pixels, including
whatever else is on screen, at whatever your current font and
theme happen to be. `pygmentize` + an HTML-to-image step is a
brittle two-tool dance with font fallbacks. Web tools like
`carbon.now.sh` round-trip your code through a third-party
server — a non-starter for unreleased internal code. `silicon`
is the only one that is *all of*: offline, scriptable,
deterministic across machines (bundled fonts + themes), and
piped from stdin so it composes with `git diff`, `pbpaste`,
`grep`, etc. v0.5.3 added a `--code-right-pad` option (clean
asymmetric layouts for diff-style snippets) and pulled in a
fixed `time-rs` so reproducible-build pipelines stay green.
