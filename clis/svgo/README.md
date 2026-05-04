# svgo

> **Plugin-based SVG optimizer that strips editor cruft, collapses
> redundant geometry, and rewrites paths into the smallest legal
> SVG that still renders identically.**
> Pinned to **v4.0.1**
> ([LICENSE](https://github.com/svg/svgo/blob/main/LICENSE), MIT).

Source: <https://github.com/svg/svgo>

## TL;DR

`svgo` is a Node.js library + CLI that parses an SVG into an AST,
runs an ordered pipeline of small transform plugins (each one an
AST → AST function), and serialises back. The default preset
(`preset-default`) bundles ~30 safe plugins — drop XML comments,
strip editor metadata (`sodipodi:`, `inkscape:`, `<sketch:*>`),
remove hidden + zero-opacity elements, merge nested `<g>` wrappers,
inline `<defs>` referenced once, round path coordinate precision,
collapse `M0 0 l5 5` → `M0 0l5 5`, and so on. SVGs exported from
Figma / Illustrator / Inkscape typically shrink **40–80%** with no
visible diff. The CLI handles single files, glob lists, recursive
directories, and stdin/stdout streaming, and the Node API is the
core — every "optimise the icons in our build" tool in the JS
ecosystem ([`vite-plugin-svgr`], [`@svgr/cli`], `gatsby-plugin-svgr`,
Next.js `next/image` SVG loader, esbuild SVG plugins) calls
`svgo`'s `optimize()` under the hood.

## Install

```bash
# global CLI (npm / yarn / pnpm / bun)
npm install -g svgo                # 4.0.1
pnpm add -g svgo
bun add -g svgo

# per-project (preferred for build pipelines — pin in package.json)
pnpm add -D svgo
npx svgo --version                  # 4.0.1

# Homebrew
brew install svgo

# verify
svgo --version
```

`svgo` is pure JS and ships its own Node-based binary; no native
build, no system libraries, no Python, no `node-gyp`. Works on any
Node ≥ 18.

## License

MIT — see
[LICENSE](https://github.com/svg/svgo/blob/main/LICENSE).
Permissive: keep the copyright + permission notice; no patent
clause, no copyleft, safe to bundle into closed-source build
tooling, npm packages, and CI containers.

## One Concrete Example

Given a typical Figma export `icon.svg` (23 KB, full of editor
metadata, `id="..."`, `data-name="..."`, fixed-precision floats):

```bash
# 1. one-shot: write icon.min.svg next to it
svgo icon.svg -o icon.min.svg
# icon.svg: 23.4 KiB - 71.2% = icon.min.svg: 6.7 KiB

# 2. recurse a directory of icons in place (overwrite originals)
svgo -rf src/icons

# 3. stream — drop into any pipe, no temp file
cat icon.svg | svgo --input=- --output=- > icon.min.svg

# 4. data-URI for inlining into CSS (base64 / unenc / enc)
svgo icon.svg --datauri=base64 -o icon.b64.txt

# 5. preview which plugins fired and how much each saved
svgo icon.svg --show-plugins -o icon.min.svg

# 6. machine-readable JSON for a CI bytes-budget gate
svgo icon.svg --output=- --multipass | wc -c
```

Tighten the default policy with a project config — `svgo.config.mjs`:

```js
// svgo.config.mjs — repo-wide policy for every build pipeline
export default {
  multipass: true,                  // re-run plugins until no shrink
  floatPrecision: 2,                // round path coords to 0.01 user units
  plugins: [
    {
      name: 'preset-default',
      params: {
        overrides: {
          // keep viewBox so the SVG scales — never strip it
          removeViewBox: false,
          // do NOT touch IDs we reference from CSS / <use href="#x">
          cleanupIds: { remove: false, minify: false },
          inlineStyles: { onlyMatchedOnce: false },
        },
      },
    },
    'removeXMLNS',                  // safe inside HTML <body>, not as standalone .svg
    { name: 'prefixIds', params: { prefix: 'icn' } },  // namespace IDs across many inlined sprites
    'sortAttrs',                    // deterministic output → smaller gzip + cleaner git diff
  ],
};
```

```bash
# CI gate: optimise the whole icons tree, fail the build if any file grew
svgo -rf src/icons \
  || { echo "svgo failed"; exit 1; }
git diff --stat src/icons | grep -E '\+\d' && {
  echo "icons larger after svgo — investigate"; exit 1;
}
```

## Niche It Fills

**Pre-publish optimisation of vector assets in a JS-ecosystem
build pipeline.** Vector editors (Figma, Illustrator, Inkscape,
Sketch) emit SVGs full of editor-specific metadata, `<defs>`
referenced once, default-value attributes, full-precision
coordinates, and pretty-printed whitespace — none of which the
browser needs. Hand-cleaning is tedious and breaks every time a
designer re-exports. `svgo` is the deterministic, plugin-pinned
machine pass that turns "designer export" into "production-ready"
without a human in the loop, and is the de-facto pre-step that
every React / Vue / Svelte / Astro SVG-to-component pipeline
([`@svgr/cli`], `vite-plugin-svgr`) calls before turning the SVG
into a JSX component.

## Why use it

1. **AST transforms, not regex.** Every plugin operates on a
   parsed XML tree, so it cannot accidentally damage attribute
   values, CDATA, nested `<style>` blocks, or scripted SVGs.
   You can write a custom plugin in ~15 lines (`fn: (ast,
   params, info) => { ... }`) and slot it in beside the
   built-ins; the same code path runs in CLI and in Node-API
   callers like `@svgr`.
2. **The default preset is conservative on purpose.** Plugins
   known to break some renderers (`removeViewBox`, `removeXMLNS`,
   `convertShapeToPath`-on-`<rect rx>`-with-CSS) are off by
   default; turning them on is an explicit per-repo decision in
   `svgo.config.mjs`. The rule of thumb is "the default preset
   is safe to run unattended in CI on a directory of designer
   exports".
3. **Round-trips through git cleanly.** `multipass: true` +
   `sortAttrs` + a fixed `floatPrecision` give byte-identical
   output for byte-identical input across machines and Node
   versions, so committing the optimised SVGs alongside the
   source SVGs (a common pattern when `<img src="icon.min.svg">`
   is referenced from prod) gives stable `git diff`s on
   re-export instead of a wall of whitespace + ID-renumbering
   noise.

For an LLM-CLI workflow, `svgo file.svg --output=- --pretty=false`
returns a single deterministic line per SVG that an agent can
diff, hash, or feed back into a sprite-generator without
re-parsing XML itself.

## Vs Already Cataloged

- **Vs [`oxipng`](../oxipng/) and [`gifski`](../gifski/) /
  [`gifsicle`](../gifsicle/):** different formats, complementary
  step in the same image pipeline. `svgo` handles vector SVG;
  `oxipng` handles lossless PNG bit-for-bit recompression;
  `gifski` / `gifsicle` handle GIFs. A typical `Makefile` for
  a static-site build runs `svgo -rf src/icons && oxipng -r
  src/img && gifsicle -O3 ...` as three separate passes — each
  tool owns one format and refuses to touch the others.
- **Vs [`pngquant`](../pngquant/):** different format
  (raster-PNG palette quantisation) and *lossy* by design.
  `svgo` is lossy in a different sense (rounding `floatPrecision`
  drops sub-pixel detail) but the diff is invisible at any
  reasonable display DPI; `pngquant` reduces a 24-bit PNG to an
  8-bit palette and changes pixel colours. They sit at different
  points in the asset pipeline and never compete for the same file.
- **Vs editor-built-in "Optimise SVG" buttons (Figma "Export →
  SVG (clean)", Inkscape "Save As → Optimised SVG"):** those
  bake one fixed policy into a GUI click, can't be CI-gated, and
  vary across editor versions. `svgo` is the same code path on
  every developer's laptop and in CI, configured once in a
  committed `svgo.config.mjs`.
- **Vs hand-rolled `xmllint` / `xmlstarlet` cleanups:** XML CLI
  tools don't know that `removeUselessStrokeAndFill` requires
  reading the cascade, that `convertPathData` requires a path
  parser, or that `inlineStyles` requires CSS specificity
  resolution. `svgo`'s plugin set is years of accumulated
  per-renderer knowledge; reproducing it with generic XML
  surgery is a category error.

## Caveats

- **`removeViewBox` is off by default for a reason.** Stripping
  `viewBox` saves a few bytes but removes the SVG's intrinsic
  scaling — the same icon then renders at a fixed pixel size
  inside a flexible container instead of scaling with CSS
  `width: 100%`. If you enable it, you must guarantee every
  consumer sets `width` + `height` explicitly. The default
  preset keeps `viewBox` and removes the redundant `width` /
  `height` attributes instead, which is the inverse trade-off
  most icon pipelines want.
- **`cleanupIds` rewrites IDs by default.** If your CSS
  selects `#icon-arrow path` or your JS does
  `document.getElementById('icon-arrow')` against the inlined
  SVG, the optimised file silently breaks because the IDs are
  now `a`, `b`, `c`. The fix is `cleanupIds: { remove: false,
  minify: false }` in the override block (or use `prefixIds`
  to namespace instead of strip).
- **`inlineStyles` resolves CSS but does not resolve external
  stylesheets.** A `<link rel="stylesheet">` inside an SVG (rare
  but legal) is left alone; only embedded `<style>` blocks are
  inlined. SVGs that depend on a host-page stylesheet must keep
  `inlineStyles` off.
- **`multipass: true` is not free.** Re-running the full plugin
  pipeline until no further shrink occurs takes 2–3× the wall
  time of a single pass; on a 5000-icon tree this is the
  difference between 4 s and 12 s. Worth it for production
  builds; skip in `--watch` mode.
- **Node-only — no native binary.** `svgo` is JavaScript; you
  need Node on the build host. There is no Rust / Go / C
  port that hits the same plugin coverage. For a non-Node
  environment, the only realistic path is to call the Node
  binary in a container or skip vector optimisation entirely.
