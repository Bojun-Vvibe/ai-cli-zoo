# gowall

- **Repo:** https://github.com/Achno/gowall
- **Version:** v0.2.4 (latest stable)
- **License:** MIT ([LICENSE](https://github.com/Achno/gowall/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install gowall` · `paru -S gowall-bin` (AUR) · `nix-shell -p gowall` · `go install github.com/Achno/gowall@latest` · static binaries on the GitHub release page (`gowall-amd64-darwin.tar.gz`, `gowall-arm64-darwin.tar.gz`, `gowall-amd64-linux.tar.gz`, `gowall-arm64-linux.tar.gz`)

## What it does

`gowall` is a single-binary CLI that re-colors images to match a named
color palette. Run `gowall convert wallpaper.jpg -t catppuccin` and it
reads the input image, maps every pixel to the nearest color in the
Catppuccin palette using a configurable algorithm
(`--algorithm nearest|riemersma|floyd-steinberg`, the last two being
true error-diffusion ditherers, not just nearest-color quantization),
and writes the result. About 30 ricer-grade palettes ship built-in
(Catppuccin all four flavors, Gruvbox, Nord, Tokyo Night, Dracula,
Solarized, Rosé Pine, Everforest, Monokai, Material Ocean, the
Wal/Pywal pasteurized variants, etc.) and you can declare your own in
`~/.config/gowall/config.yml` as a list of hex colors. Beyond palette
conversion the same binary handles the adjacent ricer-image chores
that used to need a half-dozen ImageMagick incantations:
`gowall extract wallpaper.jpg` pulls a representative palette out of
an image (k-means in CIELAB, output as hex + a preview swatch image),
`gowall pipeline image.jpg --pipeline grayscale,blur=10,brighten=20`
chains operators (resize, crop, rotate, flip, blur, sharpen, brighten,
contrast, invert, grayscale, sepia, pixelate, border, shadow,
upscale via Real-ESRGAN if installed) into one read/write pass,
`gowall mosaic *.jpg --rows 3 --cols 4` builds a contact sheet,
`gowall qrcode "https://..."` renders a QR with palette colors,
`gowall convert ascii.txt --background tokyo-night` turns ASCII art
into a styled PNG, and `gowall convert *.jpg -t nord -d ./out` runs
the whole thing across a directory in parallel. Everything is done
locally; there is no network call and no model dependency unless you
opt into the optional Real-ESRGAN upscaler.

## When to pick it / when not to

Pick `gowall` when you keep a wallpaper / screenshot / theme-asset
collection and you want every image to read in the same palette as
your shell, editor, and window manager — a one-line `gowall convert
~/Pictures/wall.jpg -t catppuccin` is the entire workflow, no GIMP
script, no Pillow notebook, no per-image tweaking. The error-diffusion
algorithms (`--algorithm floyd-steinberg`, `--algorithm riemersma`)
matter here: nearest-color quantization on a photographic image
produces visible color banding, while Floyd-Steinberg dithering
preserves perceptual gradients while still sticking to the requested
palette. Also pick it when you need an image-side tool for a ricing
setup that already standardizes on a palette ([`pywal`](#) generators,
[`stylua`](../stylua/)-formatted theme configs, Catppuccin/Nord ports
for everything) — `gowall extract` gives you the palette to feed back
into theme generators, and `gowall pipeline` covers the
"shrink + blur + add a colored border" lock-screen-wallpaper
preparation step that otherwise sends you back to ImageMagick.
Skip it if you only need general-purpose image editing
(stick with `magick` / GIMP / Affinity), if your target palette has
hundreds of colors (the visual win over a normal sRGB image is
small), or if you need format conversions outside JPEG / PNG / WEBP /
BMP / TIFF / GIF / ICO (gowall does not currently target HEIC / AVIF
input). It is a focused tool: re-color and re-shape images on the
command line. The author updates it actively (the 0.2.x series in
2025–2026 added the pipeline operator, the QR renderer, and the
Real-ESRGAN upscaler hook).

## Vs already cataloged

- **Vs [`chafa`](../chafa/) / [`viu`](../viu/):** orthogonal. Chafa
  and viu *render* an image in a terminal (Sixel, Kitty graphics,
  Unicode blocks) for viewing; `gowall` *transforms* an image and
  writes a new file. Pipe them together: `gowall convert wall.jpg
  -t nord && chafa wall.jpg`.
- **Vs [`gifsicle`](../gifsicle/):** orthogonal. Gifsicle does
  GIF-only frame surgery; gowall does palette mapping and pipeline
  operators on still images.
- **Vs [`silicon`](../silicon/) / [`freeze`](../freeze/):** different
  niche. Those render *source code* to PNG for tweet/blog use;
  gowall recolors arbitrary images. The overlap is purely thematic
  (both produce stylized PNGs).
- **Vs [`oxipng`](../oxipng/):** complementary. Oxipng losslessly
  re-encodes PNGs to be smaller; gowall changes the actual pixels.
  A common pipeline is `gowall convert ... | oxipng -` for a
  recolored, optimally-encoded output.

## Caveats

- **Quality of the recolor depends heavily on `--algorithm`.** The
  default nearest-color is fast and produces poster-style output that
  works for graphic illustrations and bad for photos. Use
  `--algorithm floyd-steinberg` (or `riemersma` for fewer artifacts
  on smooth gradients) for any photographic input.
- **The Real-ESRGAN upscale operator is not bundled.** It shells out
  to a `realesrgan-ncnn-vulkan` binary you install separately; the
  pipeline step fails fast with a clear error if it is missing.
- **No HEIC / AVIF input** as of the v0.2.x line. Convert to PNG /
  JPEG first (`magick in.heic out.png`) and feed that into gowall.
- **The built-in palette names are case-sensitive** and occasionally
  rotate as the upstream theme projects rename flavors. Pin both the
  gowall version and the palette name in dotfile setups.
- MIT ([LICENSE](https://github.com/Achno/gowall/blob/main/LICENSE))
  — permissive; safe to vendor the binary in dotfile-bootstrap
  scripts and to ship recolored output anywhere the original image
  license allowed.
