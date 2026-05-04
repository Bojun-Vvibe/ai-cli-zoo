# browsh

> **Terminal-based modern web browser** — a Go CLI that
> launches a headless Firefox in the background, drives it
> over the WebExtensions Marionette protocol, and renders
> the resulting page (full HTML / CSS / JS / WebGL / video)
> back to the terminal as Unicode half-blocks for graphics
> + real text for selectable copy/paste — meaning sites
> that need a real JS engine (modern SPAs, GitHub's
> JS-rendered pages, Slack's web UI, Google Docs viewer)
> are usable from a `tmux` pane on a remote SSH session
> without X forwarding — pinned to **v1.8.3** (commit
> [`cb3ddd53`](https://github.com/browsh-org/browsh/commit/cb3ddd533fcdaa5c4caf87154a5b627b26d38ae7),
> [LICENSE](https://github.com/browsh-org/browsh/blob/v1.8.3/LICENSE),
> LGPL-2.1-only).

Source: <https://github.com/browsh-org/browsh>

## TL;DR

A real browser whose viewport is the terminal cell grid.
Unlike `lynx` / `w3m` / `elinks` (which parse HTML and
render a text-only approximation, no JS, no CSS layout),
browsh runs an actual Firefox in headless mode, lets it
execute JS and lay out CSS as it would on a desktop, then
streams the rendered framebuffer back as colored Unicode
characters. Press `Ctrl+L` to focus the URL bar, type a
URL, and `youtube.com` opens — with the video frames
rendering as blocky half-cells and the page's JS-driven
controls fully functional.

The killer property is **JS-required sites work over SSH
on a 9600-baud link**. The remote box runs Firefox; the
SSH connection only carries the rendered Unicode
framebuffer (KB-class), not the raw HTML/JS/CSS bundles
(MB-class). Browsing GitHub, reading a Confluence page, or
clicking through a SaaS admin console from a bastion
host — all without setting up X11 forwarding, VNC, or a
remote browser-in-browser proxy.

## Install

```bash
# prebuilt binaries (Linux / macOS, x86_64 + arm64)
curl -sSfL \
  https://github.com/browsh-org/browsh/releases/download/v1.8.3/browsh_1.8.3_linux_amd64 \
  -o ~/.local/bin/browsh
chmod +x ~/.local/bin/browsh

# Homebrew
brew install browsh

# Arch Linux (AUR)
yay -S browsh-bin

# Snap
sudo snap install browsh

# Docker (zero local Firefox install)
docker run --rm -it browsh/browsh

# Build from source
git clone --branch v1.8.3 https://github.com/browsh-org/browsh
cd browsh
go build -o browsh ./interfacer
sudo install -m 0755 browsh /usr/local/bin/

# verify
browsh --version    # v1.8.3
```

Requires **Firefox ≥ 57** on `$PATH` (browsh launches it as
a child process); the Docker image bundles Firefox so the
container path needs nothing on the host.

## Example usage

```bash
# launch the TUI; opens a default start page
browsh

# go straight to a URL
browsh --startup-url https://github.com/browsh-org/browsh

# headless screenshot mode (no TUI, write a PNG and exit)
browsh --raw-text-mode https://example.com > example.txt
browsh --dump-html https://example.com > example.html

# in-TUI hotkeys
#   Ctrl+L            focus URL bar
#   Ctrl+T            new tab
#   Ctrl+W            close tab
#   Ctrl+Tab          cycle tabs
#   PageUp/PageDown   scroll a page
#   arrow keys        scroll a row/column
#   Backspace         back
#   Shift+Backspace   forward
#   Ctrl+R            reload
#   Ctrl+Q            quit (also closes the headless Firefox)
#   /                 in-page find
#   m                 toggle "monochrome" (ASCII-only fallback)

# point browsh at an existing Firefox instead of spawning one
browsh --firefox.path=/usr/bin/firefox

# use a remote browsh-server (separate Firefox host, thin client)
browsh --http-server-mode    # on the host with Firefox
# then from a thin client:
browsh --use-existing-ws-connection ws://server:4333
```

Common flags:

- `--startup-url URL` open this URL on launch
- `--firefox.path PATH` use a non-default Firefox binary
- `--firefox.with-gui` show the Firefox window too (debug)
- `--raw-text-mode URL` print page text and exit (no TUI)
- `--dump-html URL` print rendered DOM HTML and exit
- `--http-server-mode` run browsh as a server, accept thin
  clients
- `--time-limit SECS` exit after N seconds (CI / smoke tests)
- `--debug` verbose logging
- `--version` / `--help`

## Why it matters

- **JS-required sites over SSH.** Most modern web apps
  (GitHub Issues' newer pages, Confluence, Jira, Slack web,
  most internal SaaS admin consoles) render content via
  client-side JS and are unusable in `lynx` / `w3m` /
  `elinks` — the page comes up blank because the text-mode
  browsers do not run JS. browsh delegates rendering to a
  real Firefox and forwards the framebuffer, so JS-required
  sites work in a `tmux` pane on a bastion.
- **Bandwidth-thin remote browsing.** The SSH tunnel
  carries only the rendered Unicode cells (~tens of KB per
  page) instead of the raw assets (~MB). On a slow link
  (mobile tether, satellite, congested VPN), browsh on the
  far end of `ssh` is faster than running a local browser
  fetching the same page over the slow link.
- **`--raw-text-mode` and `--dump-html` are scriptable.**
  Pipe a JS-rendered page into `awk` / `jq` / `pup` /
  [`xq`](../xq/) for scraping workflows where the target
  *requires* JS execution and `curl | jq` would only see
  the empty SSR shell. `browsh --raw-text-mode URL` is the
  modern `lynx -dump` for the JS web.
- **Docker image needs nothing on the host.** `docker run
  --rm -it browsh/browsh` works on any host with Docker —
  no Firefox install, no Go runtime, no fonts to manage.
  Useful as a one-shot rescue browser on a hardened box.
- **Selectable text.** Because browsh renders *real text*
  for the text portions (not characters-as-pixels),
  `tmux` / terminal selection / clipboard integration
  work as on any text TUI — copying a URL or an error
  message out of a rendered SaaS page is one mouse drag,
  not a screenshot OCR.

## Vs Already Cataloged

- **Vs [`bombadillo`](../bombadillo/):** orthogonal — bombadillo
  is the smol-net browser (Gemini, Gopher, Finger, Spartan,
  local files), targets the alternative non-JS protocols by
  design, and renders small. browsh is the
  big-web JS-rendering browser, targets the modern HTTP
  surface, and renders pixel-ish. Pair them: bombadillo
  for `gemini://` and personal capsules, browsh for the
  JS-required corporate intranet on the other side of the
  bastion.
- **Vs `lynx` / `w3m` / `elinks`:** same TUI form factor,
  fundamentally different engines. The text-mode browsers
  parse HTML directly and render text — fast, tiny, no
  JS. browsh wraps Firefox — slower startup (Firefox cold
  start), much larger memory, but works on JS-required
  sites the others render blank. Use `lynx` / `w3m` for
  static docs, MDN, Wikipedia mirrors; use browsh for
  anything that needs JS to render.
- **Vs `carbonyl`:** very similar concept (Chromium-based
  terminal browser), different parent engine. browsh wraps
  Firefox via WebExtensions; carbonyl wraps a headless
  Chromium with a custom rendering backend. Pick by which
  upstream's behavior you prefer; browsh has the longer
  track record.
- **Vs `headless-chrome` / `puppeteer` / `playwright`:**
  same "real browser, no GUI" capability but those are
  programmable libraries (Node / Python APIs), not
  interactive TUIs. Use puppeteer / playwright when
  scripting a workflow; use browsh when *interacting*
  with a JS-required site by hand from a terminal.
- **Vs X11 forwarding (`ssh -X firefox`) / VNC / RDP:**
  graphical paths that require X / VNC infrastructure on
  both ends and 1–10 MB/s bandwidth for fluid use. browsh
  works on a plain `ssh` connection at a fraction of the
  bandwidth and survives flaky links because the protocol
  is character-cell updates, not pixel deltas.

## License

LGPL-2.1-only — see
[LICENSE](https://github.com/browsh-org/browsh/blob/v1.8.3/LICENSE).
The browsh binary is LGPL — using it from any script,
container, or remote workflow is unrestricted. Linking
browsh's libraries into a closed-source distribution is
permitted under LGPL terms (dynamic linking +
replaceability). The bundled WebExtension component is
distributed under the same license.

## Caveats

- **Firefox is a hard dependency.** browsh launches
  Firefox as a child process — install path /
  compatibility issues on the host directly affect
  browsh. The Docker image (`docker run --rm -it
  browsh/browsh`) sidesteps this by bundling its own
  Firefox.
- **Memory footprint is "Firefox-class."** browsh adds a
  small Go process on top of an already-running headless
  Firefox; expect 300–800 MB RSS depending on the page.
  This is the price of a real JS engine — `lynx` /
  `w3m` are 10–20 MB. Plan accordingly on small VMs.
- **Color depth / font / cell aspect ratio matter.** The
  half-block rendering looks crisp in a 24-bit-color
  terminal with a square cell font (e.g. `kitty`,
  `wezterm`, `alacritty`); 8-color terminals or
  non-square fonts produce a chunkier image. The
  default `xterm-256color` profile inside `tmux` works
  fine; legacy `xterm` profiles less so.
- **Mouse interaction is partial.** Click works, drag /
  hover sometimes does not — depends on terminal
  mouse-mode capability and the page's event model.
  Keyboard navigation is the reliable path; mouse is
  best-effort.
- **Last upstream release was 2024-09-27 (v1.8.3).**
  browsh's `master` continues to see commits but releases
  are infrequent. The CLI surface and protocol have been
  stable across the v1.x line; this is "stable not
  stalled" in the same sense as `playerctl` and
  `nethogs`. Watch for a future v2 that may move to a
  different headless backend.
- **Some sites detect headless Firefox and block it.**
  The same anti-bot heuristics that block `puppeteer` /
  `playwright` will block browsh — the underlying
  Firefox is genuinely headless and advertises itself as
  such by default. For non-bot interactive use, this is
  rarely an issue; for scraping, expect occasional
  captcha walls.

## As of

2026-05-04. Upstream tag `v1.8.3` (2024-09-27). browsh is in a
"stable not stalled" phase — the v1 CLI surface and the
WebExtensions Marionette protocol it depends on are stable;
re-verify if a future v2 release moves to a Chromium backend
or reshapes the headless-Firefox bridge.
