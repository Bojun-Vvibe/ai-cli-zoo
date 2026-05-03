# figlet

- **Repo:** https://github.com/cmatsuoka/figlet
- **Version:** 2.2.5 (latest stable; the canonical reference C release)
- **License:** BSD-3-Clause ([LICENSE](https://github.com/cmatsuoka/figlet/blob/master/LICENSE))
- **Language:** C
- **Install:** `brew install figlet` · `apt install figlet` · `pacman -S figlet` · `dnf install figlet` · static build from source: `make && make install` (single C file, no external deps) · the modern Go-port [`go-figure`](https://github.com/common-nighthawk/go-figure) is library-only and not the CLI

## What it does

`figlet` ("Frank, Ian, and Glenn's letters") turns a string of input text into multi-line ASCII-art banners by composing per-character glyph blocks read from a `.flf` font file. Run `figlet hello` and you get the familiar block-letter `hello` rendered in the default `standard.flf` font; run `figlet -f slant 'release v2'` and you get the same text in italic 3-D ASCII; run `figlet -f banner -w 200 'BUILD GREEN'` and you get a wide blocky CI banner sized to a 200-column terminal. The font format is a plain-text spec — each `.flf` is a small text file declaring a per-glyph height, baseline, and the literal ASCII rows that make up each printable character — so the figlet world has accumulated several hundred fonts over its 30-year life (the [figlet-fonts](http://www.figlet.org/fontdb.cgi) DB lists ~430). The CLI's surface is small but well-shaped: `-f <font>` selects the font, `-w <cols>` sets the output width (figlet wraps long inputs at word boundaries), `-c` / `-l` / `-r` align center / left / right within the width, `-k` / `-S` / `-W` control glyph kerning vs smushing vs full-width spacing, `-d <dir>` adds a font directory, and `figlet -I2` prints the path of the font directory it would search so package layouts are diagnosable. Input on stdin is supported (`echo "build $TAG" | figlet`), which is the form most CI / motd / release-script use looks like in practice. The output is plain printable ASCII (or, with `-A`, `-c` etc., aligned within an explicit column) so it composes with `tee`, `lolcat`, `boxes`, `cowsay`, terminal-color pipes, and the rest of the toy/CLI-decoration ecosystem; `toilet` is the modern rewrite that adds Unicode-color glyphs but keeps `.flf` font compatibility, so a font you author once works in both.

## When to pick it / when not to

Pick `figlet` when a build script, deploy log, MOTD, release-notes header, demo terminal, conference-talk shell, or CI summary needs an unmistakable visual marker that survives copy-paste, log aggregators, plaintext email, and screenshot OCR. The classic uses: a CI step that prints a giant `figlet "DEPLOY PROD"` before the prod stage so the operator cannot miss the boundary in scrollback; a `motd` script that greets every SSH session with the host's role in 4-line block letters; a release-notes template that opens with `figlet "v2.10.0"` so the changelog file has a clear visual anchor; a workshop terminal that prints `figlet "Lab 3"` between exercises so attendees can find their place in the recording. Pair with [`boxes`](../boxes/) to wrap the figlet output in an ASCII box; pair with [`lolcat`](https://github.com/busyloop/lolcat) (not in the catalog) to colorize the output through ANSI; pair with [`cowsay`](https://github.com/cowsay-org/cowsay) for the full toy-pipe; pair with [`tspin`](../tspin/) when the figlet banner is a separator inside a long log stream you are color-highlighting. Pair with [`gum`](../gum/) `gum style` for cases where you want a styled box with a real title bar — gum is the modern terminal-decor tool, figlet is for the specific job of "make the text *itself* huge in plain ASCII".

Skip figlet when the target output is not a fixed-width terminal (e.g. HTML email, Slack, web UI — use real typography); when the output needs to be screen-reader-friendly (figlet ASCII art is hostile to accessibility tools); when you need Unicode coverage or color glyphs (use `toilet`, the descendant project, instead — `apt install toilet`); when the consumer is a structured-log aggregator that will index every character cell as a separate token (you will pollute the log index with thousands of `_` and `|` symbols); and when a Twelve-Factor "logs are streams of events" purist will revert your PR (in that case keep banners to interactive shells only, not service stdout).

## Vs already cataloged

- **Vs [`boxes`](../boxes/):** complementary. `boxes` draws an ASCII border *around* arbitrary text; figlet renders the text itself as block letters. The classic combo is `figlet "RELEASE" | boxes -d stone` for a bordered banner.
- **Vs [`gum`](../gum/) (`gum style`):** different era, different intent. `gum style --border double --padding "2 4" "Release v2.10.0"` gives you a polished modern terminal card with Unicode borders and ANSI color, sized for one line. figlet gives you 6-row ASCII glyphs of the text itself, no Unicode required, copy-paste-stable into any plaintext context. Use gum for interactive prompts and styled UI, figlet for the "my CI log needs a giant DEPLOY marker" case.
- **Vs [`glow`](../glow/):** orthogonal. glow renders Markdown to a styled terminal view; figlet renders one short string to giant ASCII art. They share zero scope.
- **Vs [`vhs`](../vhs/) / [`asciinema`](../asciinema/):** orthogonal. Those record terminal sessions. figlet produces content that *appears in* such recordings — the conference-demo banner you see in the recorded `.cast` is almost always a `figlet` line.
- **Vs `toilet`:** `toilet` is a from-scratch GPL-2.0 rewrite by the same author who wrote `cacaview`. It is `.flf`-compatible (most figlet fonts work in toilet), supports Unicode glyphs (`--metal`, `--gay`), and adds color filters. If your terminal has Unicode + truecolor support and you want fancy, install toilet alongside figlet. Most distros ship both.

## Caveats

- **`figlet 2.2.5` is the last release of the canonical C reference codebase** (2012). The project is mature, not actively developed; the maintained mirror is [cmatsuoka/figlet](https://github.com/cmatsuoka/figlet) but no breaking changes are expected and none are needed — the format and CLI flags have been stable for 30 years. Several Linux distros ship slightly patched builds; the upstream tarball is the source of truth.
- **Default output is monochrome ASCII only.** No Unicode, no ANSI color. Pipe through `lolcat` for color, or use `toilet --gay` for built-in coloring. Most terminals render figlet output as plain text in the default color.
- **Width matters more than people think.** `-w 80` is the safe default for terminals; on a narrow terminal (50 cols) and a wide font (`-f banner`) the output wraps mid-glyph and becomes unreadable. CI logs in some viewers default to 132 cols, others to 80 — pin `-w` explicitly in scripted use.
- **The font path is install-dependent.** `brew install` puts fonts under `$(brew --prefix)/share/figlet/fonts`; `apt install figlet` puts them under `/usr/share/figlet/`; sourceforge and the [figlet font DB](http://www.figlet.org/fontdb.cgi) ship hundreds more, dropped into `~/.figlet/`. `figlet -I2` prints the actual directory it will search.
- **Do not rely on figlet output as a structured log marker.** The newline / spacing structure makes log aggregators see one banner as 6 separate log lines (or one giant line, depending on the parser). Use it for human eyes in interactive scrollback or in well-known release / motd / boundary-marker spots, not as a substitute for a real `level=info event=deploy` field.
- BSD-3-Clause ([LICENSE](https://github.com/cmatsuoka/figlet/blob/master/LICENSE)) — permissive; safe to bundle into release scripts, Docker base images, and CI runner images. Note that some `.flf` font files have their own licenses inside the file header; redistribute with attribution if you ship fonts as part of your tooling.

## Example invocations

```bash
# Install
brew install figlet                 # macOS
sudo apt install figlet             # Debian / Ubuntu
sudo pacman -S figlet               # Arch

# Default font, default width
figlet hello

# Pick a font
figlet -f slant 'release v2.10.0'
figlet -f banner 'DEPLOY PROD'

# Constrain to a known terminal width
figlet -w 80 -c 'BUILD GREEN'       # centered in 80 cols

# Read from stdin (the CI / shell-script form)
echo "deploy ${TAG}" | figlet -f standard

# List the font directory figlet is searching
figlet -I2

# List installed fonts
ls "$(figlet -I2)" | head -20

# Use a font dropped into a custom directory
figlet -d ~/.figlet -f bigmoney-ne 'PAYDAY'

# Compose with boxes for a bordered banner
figlet 'RELEASE' | boxes -d stone

# CI / deploy-script idiom: clear visual boundary between stages
figlet -w 100 -c "STAGE: ${1:-build}"
```
