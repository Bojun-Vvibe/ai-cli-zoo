# imv

> **Image viewer for X11 / Wayland with vim keybinds and a real
> CLI** — a single C binary that opens images in a borderless,
> tiling-WM-friendly window with `hjkl` navigation, programmable
> bindings (`~/.config/imv/config`), animation support (GIF /
> APNG), and a stdin-pipe + IPC surface so it integrates into
> shell pipelines and editor workflows the way most GUI viewers
> never do — `imv -` reads paths from stdin (`fd '\.png$' | imv
> -`), `imv-msg` sends commands to a running window (`imv-msg
> $(pgrep imv) next`), and the on-screen overlay (`d`) shows
> filename + dimensions + scale + index for triage workflows.
> Pinned to **v5.0.1** (commit
> `3ef6039b9a13ac9ba842bf6512c9f700c16571e3`,
> [LICENSE](https://git.sr.ht/~exec64/imv/tree/v5.0.1/LICENSE),
> MIT).

Source: <https://git.sr.ht/~exec64/imv> (mirror:
<https://github.com/eXeC64/imv>)

## TL;DR

`imv` is what you reach for on a Linux / BSD desktop where the
image viewer needs to (a) launch in <100ms from a tiling WM
keybind, (b) take its input from stdin so a `find` / `fd` / shell
glob feeds it, (c) be controllable from another process so an
editor or file manager can scrub the displayed image without
relaunching, and (d) not pull in half of GNOME / KDE as
dependencies. The native Wayland (`wlroots` / `Hyprland` /
`Sway`) and X11 backends are first-class — no XWayland fallback
required on Wayland sessions.

`imv ~/screenshots/*.png` opens a window that holds every match;
`hjkl` (or arrow keys) navigates `prev/next/zoom in/out`, `f`
toggles fullscreen, `d` toggles an on-screen overlay with
filename + dimensions + scale + index, `r` rotates 90°, `c`
centres the image, `=` / `-` zoom by one step, `[` / `]` cycle
frames in an animated GIF / APNG, `q` quits.

The piped form is where it earns its place: `fd '\.(png|jpg)$' .
| imv -` reads paths line-by-line from stdin, so a `ripgrep`
+ `fd` triage workflow ("find every PNG larger than 1MB,
preview them") is one shell pipeline. The IPC surface is the
companion `imv-msg <pid> <command>` binary — wire it into a
file-manager binding (e.g., yazi opener) so cursoring onto an
image in the file manager scrubs the existing imv window
instead of spawning a new one.

## Install

```bash
# Arch Linux
sudo pacman -S imv

# Debian / Ubuntu
sudo apt install imv

# Fedora
sudo dnf install imv

# Alpine
sudo apk add imv

# FreeBSD
sudo pkg install imv

# Build from source
git clone https://git.sr.ht/~exec64/imv
cd imv
git checkout v5.0.1
make
sudo make install

# verify
imv --version    # 5.0.1
```

Build deps: `meson`, `pkg-config`, one of `libxkbcommon` +
`wayland` (Wayland backend) or `libxkbcommon` + `libx11` +
`libxext` (X11 backend), `pango`, `cairo`, plus per-format
loaders (`libjpeg-turbo`, `libpng`, `libtiff`, `libheif`,
`librsvg`, `libnsgif`).

## License

MIT — see
[LICENSE](https://git.sr.ht/~exec64/imv/tree/v5.0.1/LICENSE).
Permissive; no attribution required for binary redistribution.

## Hot keybinds

Defaults (rebindable in `~/.config/imv/config`):

- `h` / `j` / `k` / `l` — previous / zoom out / zoom in / next
- `arrow keys` — same as `hjkl`
- `gg` / `G` — first / last image in the queue
- `f` — toggle fullscreen
- `d` — toggle on-screen overlay (filename / dimensions /
  scale / index / mtime)
- `c` — centre image in window
- `r` / `R` — rotate 90° clockwise / counter-clockwise
- `=` / `-` — zoom in / out one step
- `[` / `]` — previous / next frame in animated GIF / APNG
- `p` — toggle play/pause for animated images
- `o` — open file dialogue (run a configured opener)
- `x` — remove current image from the queue (does not delete
  from disk)
- `q` — quit
- `:` — command-mode prompt (`:open /path`, `:close`, `:bind`,
  `:exec <shell-command>`)

## Why use it

- **Native Wayland support** without XWayland — works on Sway /
  Hyprland / wlroots-based / GNOME-Wayland sessions with proper
  HiDPI, fractional scaling, and per-monitor DPI handling.
  `feh` / `sxiv` are X11-only and look blurry under XWayland.
- **Stdin pipe input.** `imv -` reads paths line-by-line so any
  shell pipeline that produces filenames is a slideshow source.
  The combinator that GUI viewers do not have:
  `fd -e jpg . | shuf | imv -` is a randomised photo browser of
  the current tree.
- **Out-of-process control via `imv-msg`.** Send commands to a
  running imv from a script: `imv-msg $(pgrep -x imv) close
  current`, `imv-msg $(pgrep -x imv) open ~/Downloads/new.png`.
  The hook for "file manager preview pane that scrubs without
  relaunching."
- **Animated image support** — GIF / APNG / animated WebP
  play with frame-by-frame controls (`[` / `]`), pause (`p`),
  and speed adjustment via `:bind <key> next_frame 100ms` style
  config.
- **Programmable bindings** — `~/.config/imv/config` rebinds any
  key to any imv command or shell command (`bind <Shift+s> exec
  cp $imv_current_file ~/keep/`).

## Vs Already Cataloged

- **Vs [`chafa`](../chafa/) / [`viu`](../viu/):** orthogonal —
  chafa / viu render images *inside* the terminal as unicode
  blocks or graphics-protocol pixels; `imv` opens a *separate
  window*. Use chafa/viu for over-SSH previews; use imv on a
  local desktop where a real window is fine.
- **Vs [`yazi`](../yazi/):** complementary — yazi previews
  images inline in its file-manager pane, but for full-screen
  inspection you bind a yazi opener key to `imv-msg $(pgrep -x
  imv) open $@` so cursoring onto a different image in yazi
  swaps the imv window contents instead of spawning a new
  process. The "file manager + dedicated viewer" pairing.
- **Vs `feh` / `sxiv` / `nsxiv`:** imv is the Wayland-native
  successor with stdin-pipe input and out-of-process IPC. Pick
  feh on legacy X11-only setups; pick imv on Wayland or when
  the IPC + pipe model matters.

## Caveats

- **Linux / BSD only.** No native macOS or Windows builds (use
  Preview.app / Photos / nomacs there). `imv` is a Wayland /
  X11 client.
- **No editing** — viewer only. Rotate is in-window display only,
  not written back to file. For a TUI that *edits* metadata,
  pair with `exiftool` or `jhead` in a shell binding.
- **Format coverage depends on which loader libs were present at
  build time.** Distro packages enable JPEG / PNG / TIFF / GIF /
  WebP / HEIF / SVG / APNG / NSGIF by default; a custom build
  may not. Run `imv --version` to see the compiled-in loader
  list, and check the docs if a `.heic` from an iPhone refuses
  to open.
- **The IPC surface uses Unix sockets keyed by PID.** Multiple
  concurrent imv windows need distinct PIDs (`imv-msg <pid>`),
  which means a script that wants to drive a *specific* window
  must remember which pid it spawned. Single-window workflows
  are the common case.
- **No video support.** GIFs / APNGs / animated WebPs animate;
  `.mp4` / `.mov` / `.webm` need `mpv` instead. Pair both.
- **Hosted on SourceHut**, not GitHub. Issue tracker and patches
  go through `lists.sr.ht` mailing lists; the GitHub mirror is
  read-only and not the place to file issues.
