# nsxiv

> **Neo Simple X Image Viewer — a tiny, keyboard-driven X11
> image browser** that opens a directory of images (or a list
> piped on stdin) into a thumbnail grid + full-view pair, with
> vim-style navigation, on-the-fly zoom / rotate / flip,
> external-script hooks (`~/.config/nsxiv/exec/key-handler`),
> animated GIF playback, and zero desktop-environment
> assumptions — fork-continuation of the abandoned `sxiv`,
> kept actively maintained at the `nsxiv/nsxiv` mirror.
> Pinned to **v34**, SPDX `GPL-2.0`,
> [LICENSE](https://github.com/nsxiv/nsxiv/blob/master/LICENSE).

Source: <https://github.com/nsxiv/nsxiv>
(canonical home: <https://codeberg.org/nsxiv/nsxiv>)

## TL;DR

`nsxiv` is the spiritual successor to `sxiv` — when the
upstream went dormant in 2021 the community forked it and kept
shipping (33 → 34 since). It is a *suckless-style* image viewer:
one C binary, no Qt / GTK / Electron, opens in milliseconds,
exits on `q`. Two modes: **image mode** (one picture filling
the window, `j`/`k` next/prev, `+`/`-` zoom, `r` rotate, `<`/`>`
gif frames) and **thumbnail mode** (`Return` toggles, arrow
keys / `gg` / `G` to navigate, `m` marks for batch ops). The
killer feature is the `key-handler` hook: any unbound key press
executes `~/.config/nsxiv/exec/key-handler <key> <files...>`
with the currently selected images on stdin — turning nsxiv
into the *front-end* for any shell pipeline (rotate with
`jpegtran`, optimise with `jpegoptim`, copy to clipboard with
`xclip`, upload, delete, tag with `exiftool`, etc.). Pure X11,
so on Wayland you need XWayland; macOS users want
[`imv`](../imv/) instead.

## Install

```bash
# Arch
sudo pacman -S nsxiv

# Debian / Ubuntu
sudo apt install nsxiv

# Homebrew (X11 via XQuartz)
brew install nsxiv

# From source
git clone https://github.com/nsxiv/nsxiv && cd nsxiv
make && sudo make install

# verify
nsxiv -v   # nsxiv 34
```

## License

GNU GPL-2.0 — see
[LICENSE](https://github.com/nsxiv/nsxiv/blob/master/LICENSE).
Copyleft, redistribute source on binary distribution.

## Representative Commands

```bash
# 1. open a single image
nsxiv photo.jpg

# 2. open a whole directory in thumbnail mode
nsxiv -t ~/Pictures/2026/

# 3. recurse into subdirectories
nsxiv -r ~/Pictures/

# 4. read file list from stdin (pair with find / fd / fzf)
fd -e jpg -e png . ~/Pictures | nsxiv -i -t

# 5. fullscreen slideshow, 3-second interval
nsxiv -f -S 3 ~/Pictures/album/

# 6. private-mode (no thumbnail cache writes), good for sensitive dirs
nsxiv -p secret/

# 7. wire a key-handler: rotate selected with `jpegtran` on `r`
mkdir -p ~/.config/nsxiv/exec
cat > ~/.config/nsxiv/exec/key-handler <<'EOF'
#!/bin/sh
case "$1" in
  r) while read -r f; do jpegtran -rotate 90 -copy all -outfile "$f" "$f"; done ;;
  d) while read -r f; do trash-put "$f"; done ;;
esac
EOF
chmod +x ~/.config/nsxiv/exec/key-handler
```

## Niche / Category

GUI image viewer (X11, suckless lineage).

## Why It Is Orthogonal

The catalogue already lists [`chafa`](../chafa/),
[`viu`](../viu/), and [`imv`](../imv/), but they are different
animals: `chafa` and `viu` render images **into the terminal**
(sixel / kitty graphics / ANSI half-blocks) — perfect for
SSH and tmux, useless for serious browsing of a 5000-photo
directory. `imv` is a Wayland-first image viewer with an IPC
control socket aimed at scripting from a compositor / status
bar. `nsxiv` fills the niche of a **pure-X11, keyboard-driven,
hook-extensible thumbnail browser**: the right tool when you
are sitting at an Xorg desktop, `cd`-ing into a folder of RAW
exports or screenshots, and want to flip through them at
60 Hz, mark a subset, and pipe the marked filenames into a
shell loop. Pairs with [`exiftool`](../exiftool/) (read /
write metadata in the key-handler), [`jpegoptim`](../jpegoptim/)
and [`pngquant`](../pngquant/) (compress in place from a
keystroke), [`gifsicle`](../gifsicle/) (post-process GIFs you
just previewed), and [`fd`](../fd/) (feed `nsxiv -i` from a
filtered file list). Reach for it when Wayland / XWayland is
fine, you live in keyboard-shortcut land, and you want a
viewer that *gets out of the way* of a shell-driven photo
workflow rather than an Electron gallery app.
