# d2

> **Text-to-diagram for the terminal age** — a single Go binary
> that compiles a small declarative language (`a -> b: label`)
> into clean SVG / PNG / PDF diagrams, with first-class layout
> engines (ELK, Dagre, the in-house TALA), live-reload watch
> mode, sketch-style hand-drawn rendering, themes, and an embed
> story for static-site generators (Hugo, Docusaurus, mdBook).
> Pinned to **v0.7.1** (commit
> `25ad4cdb8fe0e7abad9a0e710bbdb840b2c826b7`,
> [LICENSE.txt](https://github.com/terrastruct/d2/blob/v0.7.1/LICENSE.txt),
> MPL-2.0).

Source: <https://github.com/terrastruct/d2>

## TL;DR

`d2` is the missing middle between "ASCII art that breaks the
moment a label gets long" and "open Figma / draw.io and lose your
git history". You write architecture, sequence, ER, and class
diagrams as readable text (`auth -> db: query users`), check it
into the repo next to the code it documents, and `d2 input.d2
output.svg` produces a publication-grade diagram with sensible
auto-layout. Live-reload (`d2 --watch`) gives you sub-second
iteration in the browser. The language has just enough features
(containers, connections, classes, vars, imports) to model real
systems without becoming Graphviz-shaped (where layout is a
fight).

## Install

```bash
# Homebrew (macOS / Linux)
brew install d2

# Install script (any OS)
curl -fsSL https://d2lang.com/install.sh | sh -s -- --version v0.7.1

# Go install
go install oss.terrastruct.com/d2@v0.7.1

# Pre-built binary release
curl -Lo d2.tar.gz "https://github.com/terrastruct/d2/releases/download/v0.7.1/d2-v0.7.1-darwin-arm64.tar.gz"
tar xf d2.tar.gz
sudo install d2-v0.7.1/bin/d2 /usr/local/bin/

# Optional ELK layout engine (better for crowded diagrams; needs Node)
brew install node
# d2 ships ELK as bundled JS — `d2 --layout=elk` Just Works after `d2` install

# verify
d2 --version    # v0.7.1
```

## License

MPL-2.0 — see
[LICENSE.txt](https://github.com/terrastruct/d2/blob/v0.7.1/LICENSE.txt).
File-level copyleft: you can use `d2` in commercial products
without source disclosure; modifications to `d2`'s own source
files must stay MPL. (The proprietary TALA layout engine is a
separate, paid component with its own license.)

## One Concrete Example

```bash
# 1. minimal hello-world: write text, compile to SVG
cat > arch.d2 <<'EOF'
users: Users {shape: person}
lb: Load Balancer {shape: hexagon}
api: API {
  auth: Auth Service
  pay: Payments
}
db: Postgres {shape: cylinder}

users -> lb -> api.auth -> db
api.pay -> db
EOF
d2 arch.d2 arch.svg
# open arch.svg in any browser

# 2. live-reload watch mode (opens browser, refreshes on save)
d2 --watch arch.d2

# 3. sequence diagrams (auto-detected from the shape)
cat > seq.d2 <<'EOF'
shape: sequence_diagram
client -> api: POST /login {creds}
api -> db: SELECT * FROM users WHERE email=...
db -> api: row
api -> client: 200 {token}
EOF
d2 seq.d2 seq.png

# 4. theme + sketch (hand-drawn) mode
d2 --theme 200 --sketch arch.d2 arch-handdrawn.svg

# 5. choose a layout engine — dagre (default), elk, or TALA (paid)
d2 --layout=elk arch.d2 arch-elk.svg
d2 --layout=dagre arch.d2 arch-dagre.svg

# 6. embed in markdown via fmt + multi-board
cat > docs.d2 <<'EOF'
# auto-imported sub-diagrams
auth: @auth-detail
pay:  @pay-detail
EOF
d2 fmt docs.d2          # canonicalize formatting (like gofmt)

# 7. generate a PDF for stakeholders (vector, scalable)
d2 arch.d2 arch.pdf

# 8. animate a sequence of boards into a multi-frame SVG
d2 --animate-interval 1000 multi-board.d2 anim.svg

# 9. produce raw structured ASTs (handy for tools / agents)
d2 --bundle=false --check arch.d2     # check-only, lints + reports issues

# 10. CI-friendly batch render
find docs -name '*.d2' -print0 | xargs -0 -n1 -P4 -I{} \
  sh -c 'd2 "$1" "${1%.d2}.svg"' _ {}
```

## Niche It Fills

**Diagrams as a versionable text artifact, with auto-layout that
doesn't fight you.** The two existing options were (a) Mermaid /
PlantUML / Graphviz — text-based but with cramped layouts, weak
typography, and limited shape vocabulary — or (b) draw.io /
Lucidchart / Figma — beautiful but binary blobs that don't diff,
don't merge, and rot the moment the system changes. `d2` is the
first tool to match the *visual quality* of the GUI tools while
keeping the *git-friendliness* of the text tools. For an LLM-CLI
workflow, an agent can author a `system.d2` file (legible
syntax, easy to generate) and the human reviewer sees a real
diagram in CI artifacts or in the rendered docs site without any
binary asset round-trip.

## Why use it

Three things `d2` does that Mermaid / PlantUML / Graphviz do not,
that justify adding a new tool:

1. **Auto-layout that's actually good.** The default Dagre engine
   handles small graphs cleanly; ELK (bundled, free) handles
   crowded ones with proper edge routing and minimum crossings;
   TALA (paid) is best-in-class for very large architecture
   diagrams. Compare side by side: a 40-node service map in
   Graphviz is a wall of crossed lines, in Mermaid is unreadable
   below 1080p, in `d2` --layout=elk is publication-ready
   without manual coordinate fiddling.
2. **A real type system for nodes and connections.** Containers
   (`api: { auth; pay }`), classes (`classes: { server: { shape:
   hexagon; style.fill: lightblue } }`), variables, and imports
   (`auth: @auth.d2`) let you keep one source of truth across
   many diagrams — change the `server` class once, every diagram
   that uses it updates. Mermaid's per-node styling is
   per-occurrence and doesn't compose.
3. **`d2 fmt` + `d2 --watch` make iteration fast.** `d2 fmt` is
   the gofmt of diagrams: canonical whitespace, deterministic
   output, no diff noise. `d2 --watch` boots a local server that
   re-renders on save in <500 ms for typical diagrams, so the
   loop is "edit text → glance at browser tab" not "open GUI →
   wait for canvas → drag boxes". Combined, they make a diagram
   feel like code.

## Vs Already Cataloged

- **Vs [`mdbook`](../mdbook/) / [`mdcat`](../mdcat/) /
  [`glow`](../glow/):** complementary — those render Markdown;
  `d2` produces the diagrams you embed inside that Markdown.
  `mdbook` has a `d2` preprocessor; the standard pattern is `d2`
  source in the repo + `mdbook build` rendering it into the
  static site.
- **Vs [`silicon`](../silicon/) / [`freeze`](../freeze/):**
  orthogonal — those render *code snippets* into PNG/SVG for
  blog posts; `d2` renders *system diagrams*. Both produce SVG
  for the same docs site, but their input grammars (a code
  snippet vs a connection graph) and use cases (showing a
  function vs showing an architecture) don't overlap.
- **Vs Mermaid (not cataloged):** the closest peer — same
  text-to-diagram positioning, broader reach (GitHub renders
  Mermaid in markdown natively, no `d2` equivalent yet). `d2`
  wins on layout quality, multi-board / import composition,
  shape vocabulary (sketch mode, person shape, cylinder, queue,
  cloud), and `d2 fmt`. Mermaid wins on zero-install rendering
  in GitHub READMEs and the GitHub Actions ecosystem. For
  in-repo docs that build via mdBook / Hugo / Docusaurus, `d2`
  is the better choice; for a single GitHub README diagram
  someone will glance at, Mermaid is.
- **Vs PlantUML (not cataloged):** PlantUML has 15+ years of
  shape vocabulary (UML class / activity / state / component /
  deployment), Java dependency, and Graphviz under the hood for
  layout — which means crowded diagrams suffer the same edge
  routing as Graphviz. `d2` is younger and intentionally narrows
  the shape catalog (no UML state machine syntax), but the
  output looks ~2× better at the same complexity. Pick PlantUML
  if you specifically need UML formalism; pick `d2` for general
  architecture / sequence / ER diagrams.
- **Vs Graphviz (not cataloged):** Graphviz is the foundation
  layout engine, lower-level — DOT language is more powerful but
  has no notion of "container", "class", or "import". You spend
  more time in `subgraph cluster_*` boilerplate. `d2` ships
  Graphviz-style power behind a higher-level grammar.

## Caveats

- **GitHub doesn't render `.d2` natively.** Unlike Mermaid (which
  GitHub renders inline in markdown code blocks), `.d2` files
  show as plain text. You either render to SVG and commit the
  SVG alongside the `.d2`, or rely on a docs build step
  (mdBook / Hugo / Docusaurus / a custom GH Action) to turn
  source into images for the published docs.
- **TALA is paid.** The best layout engine, [TALA](https://terrastruct.com/tala),
  requires a Terrastruct license and runs as a sidecar binary.
  Free Dagre and ELK are good enough for ~95% of diagrams;
  reach for TALA only when you have 50+ nodes or strict
  publication constraints.
- **Active language — formatting drift between releases.** `d2`
  hit 1.0 in 2024 and is still evolving syntax (export semantics,
  glob imports, classes). Pin a version (`go install
  oss.terrastruct.com/d2@v0.7.1`) and keep CI on the same; running
  `d2 fmt` from a different version can produce a noisy diff.
- **Layout is heuristic, not deterministic across versions.**
  Dagre / ELK upgrades occasionally re-shuffle node positions
  even when the source file hasn't changed. SVG diffs are
  meaningful; if you commit rendered output, expect occasional
  re-render churn at upgrade time. Either commit only `.d2`
  source and render in CI, or pin both `d2` and the layout
  engine.
- **Big diagrams hit a readability ceiling that no tool fixes.**
  At 80+ nodes, even ELK + sketch mode produces a wall.
  `d2`'s answer is multi-board / import composition: split into
  per-subsystem diagrams with cross-links, not one mega-graph.
  Plan for this from the start; refactoring a 200-node
  monolithic `.d2` file is no fun.
- **No interactive editor in the binary.** `d2 --watch` is a
  re-render loop, not a GUI canvas. If a non-technical
  stakeholder needs to drag boxes, they'll still want
  draw.io / Lucid; `d2` is for engineers who write diagrams as
  text.
