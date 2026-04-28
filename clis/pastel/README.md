# pastel

- **Upstream:** https://github.com/sharkdp/pastel
- **Version:** v0.12.0 (2026-02-14)
- **License:** Apache-2.0 OR MIT — https://github.com/sharkdp/pastel/blob/master/LICENSE-APACHE and https://github.com/sharkdp/pastel/blob/master/LICENSE-MIT (SPDX: `Apache-2.0 OR MIT`)

## What it does

`pastel` is a command-line tool from the author of `bat` and `fd` for
generating, analyzing, converting, and manipulating colors. It supports
named CSS colors, hex, RGB, HSL, Lab, and LCh; can compute distance and
contrast (WCAG) between colors; and can build palettes via mixing,
gradients, sorting, or random generation. Useful for designing terminal
themes, picking accessible foreground/background pairs, and scripted
color math in shell pipelines.

## Example

```sh
# Show a color and its variations
pastel color "#ff8800"

# Build a 5-step gradient between two colors
pastel gradient "#1e3a8a" "#fbbf24" -n 5

# Pick a readable text color for a given background
pastel textcolor "#264653"
```
