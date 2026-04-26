# stagewise

> Snapshot date: 2026-04. Upstream: <https://github.com/stagewise-io/stagewise>

A **developer-facing browser with a coding agent built in**. You select a DOM
element on your running localhost app, type a change in plain English, and
stagewise opens the right source file in your editor and applies the edit.

## 1. Install footprint

- VS Code extension (`stagewise.stagewise-vscode-extension`) — the primary
  surface.
- A small toolbar `<script>` (`@stagewise/toolbar`) you drop into your dev
  HTML so the browser side knows which element you clicked.
- Framework adapters for React / Vue / Next.js / Nuxt / SvelteKit auto-inject
  the toolbar in dev mode only.
- Latest release: **`stagewise-vscode-extension@0.11.4`** (the repo is a
  monorepo; each package tags independently).

## 2. License

AGPL-3.0. See [`LICENSE`](https://github.com/stagewise-io/stagewise/blob/9d0a0321051fe85e8e2893458937f7f85a852e86/LICENSE).
The AGPL choice matters: hosted/SaaS rewrappers must publish their changes.

## 3. Models supported

Bring-your-own-key against any provider the embedded agent supports —
Anthropic, OpenAI, Gemini, OpenRouter, plus any OpenAI-compatible local
endpoint (Ollama, vLLM, LM Studio). The selected model is per-workspace,
configured in the VS Code extension settings.

## 4. MCP support

**Yes (client).** The agent can attach to MCP servers configured in the
extension; the canonical use is wiring a project-specific MCP (e.g. a
component-library docs server) so the model can ground edits in your
design system rather than hallucinating Tailwind classes.

## 5. Sub-agent model

Single agent. The interesting axis is *grounding*: every prompt comes
bundled with the selected element's outerHTML, computed styles, and a
source-map-resolved file path, so the agent doesn't need to spawn a
"locate the file" sub-task — that's already done by the toolbar before
the prompt is sent.

## 6. Telemetry stance

**Opt-in.** Anonymous usage analytics gated behind a first-run prompt
in the VS Code extension. The toolbar itself never phones home.

## 7. Prompt-cache strategy

Provider-default. The system prompt + project-config block are stable
across edits in a session, so Anthropic prefix-cache hits naturally;
the per-edit element snapshot busts the cache for the user-message
portion (intentional — that's the whole point of the request).

## 8. Hot keybinds (toolbar + extension)

| Surface | Binding | Action |
|---------|---------|--------|
| Browser toolbar | `Alt+S` (mac: `Opt+S`) | Toggle "select an element" mode |
| Browser toolbar | Click element | Lock selection, open prompt box |
| Browser toolbar | `Cmd+Enter` | Send prompt to the agent |
| VS Code | `Cmd+Shift+P` → `Stagewise: ...` | All extension commands |
| VS Code | (per-project) | View / approve diffs before save |

## 9. Killer feature, weakness, when to choose

- **Killer:** the **point-and-prompt loop**. Every other AI coder asks
  you to describe *which* button is broken; stagewise lets you click
  the actual broken button in your running app, and the source-map +
  framework adapter chain turns that click into the exact JSX/Vue
  template line the model needs to edit. For UI-iteration work this
  collapses minutes of "find the file" into zero.
- **Weakness:** AGPL-3.0 makes it awkward for some commercial closed-
  source contexts; the toolbar requires a dev-mode HTML inject so it
  can't help you on a production-only app you can't rebuild; and the
  whole value prop assumes a *web* frontend — native mobile / desktop
  is out of scope.
- **Choose it when:** you're building a web frontend, you live in
  VS Code, and the bottleneck in your AI-assisted workflow is "tell
  the model *which* element I mean" rather than "tell the model *what*
  to change."
