# cua

> Snapshot date: 2026-04. Upstream: <https://github.com/trycua/cua>

Open-source infrastructure for **Computer-Use Agents** — full-desktop control
(macOS, Linux, Windows) inside reproducible VM sandboxes. Think "the runtime
layer underneath an Operator-style agent" rather than a chat REPL.

## 1. Install footprint

- `pip install cua-computer cua-agent` for the Python SDK.
- `npm install @trycua/computer` for the TypeScript SDK.
- macOS host needs the **Lume** VM driver (`brew install trycua/tap/lume`)
  to spin up sandboxed macOS/Linux guests via Apple Virtualization.
- A pre-built Ubuntu/macOS image is pulled on first run (~5 GB).
- Latest release: **`cua-driver-v0.0.5`** (driver component; SDK packages
  version independently on PyPI / npm).

## 2. License

MIT. See [`LICENSE.md`](https://github.com/trycua/cua/blob/66bed0db7161bca000c206358ea7351bdadcb15c/LICENSE.md).

## 3. Models supported

Provider-agnostic. The agent loop is a thin wrapper around any vision-capable
LLM that can emit `click(x,y)` / `type("...")` / `key("Cmd+S")` actions:

- Anthropic (Claude with computer-use tool — first-class).
- OpenAI (GPT-4o / o-series via the Responses API computer-use preview).
- Local VLMs through any OpenAI-compatible endpoint.
- A custom `BaseAgent` interface for plugging in fine-tuned UI-grounding
  models (UI-TARS, ShowUI, etc.).

## 4. MCP support

**Yes (server side).** `cua-mcp-server` exposes the sandbox as MCP tools
(`screenshot`, `click`, `type`, `bash`, `read_file`) so any MCP client can
drive a desktop. There's no built-in MCP *client* — cua is the thing the
client talks to.

## 5. Sub-agent model

Single agent loop per `Computer` session, but the SDK is explicitly designed
to be embedded inside a larger orchestrator. The `AgentLoop` class is
swappable; the reference loop is observe → plan → act → verify with screenshot
diffs as the verification signal.

## 6. Telemetry stance

**Opt-in.** `CUA_TELEMETRY=1` enables anonymous run metrics (action counts,
model latency). Off by default. The VM is isolated from the host network
unless you pass `--shared-network`.

## 7. Prompt-cache strategy

Anthropic ephemeral cache on the system prompt + tool schema. Screenshots
are *not* cached (they change every step) — instead the loop trims old
screenshots from history after N turns to keep token cost bounded.

## 8. Hot keybinds (REPL / SDK)

cua is an SDK, not a TUI. The notable surface is the `Computer` context
manager in Python:

| Call | Action |
|------|--------|
| `async with Computer() as c:` | Boot a fresh VM, returns when SSH/VNC ready |
| `await c.screenshot()` | PNG bytes of the guest screen |
| `await c.left_click(x, y)` | Click at guest coordinates |
| `await c.type("hello")` | Synthesize keystrokes |
| `await c.bash("ls /")` | Run shell in the guest, get stdout |
| `await c.read_file(path)` | Pull file out of the VM |

The CLI side is `lume` for VM lifecycle: `lume run macos-sequoia`,
`lume ls`, `lume stop <name>`.

## 9. Killer feature, weakness, when to choose

- **Killer:** **the VM is the trust boundary.** Other computer-use stacks
  hand the model your real desktop. cua spins up a disposable macOS or
  Linux guest via Apple Virtualization (so guest macOS is *legal*, unlike
  most alternatives), runs the agent inside it, and tears it down. That
  makes the "what if the model `rm -rf`'s my home" question go away.
- **Weakness:** macOS host only for the macOS-guest path (Apple
  Virtualization is Apple-silicon-locked). Linux guests work anywhere but
  you lose the "test my real Mac app" use case. First-boot image pull is
  slow.
- **Choose it when:** you're building an agent that needs to drive a real
  desktop GUI — QA automation across native apps, end-to-end browser +
  desktop flows, or training data collection for a UI-grounding model —
  and you don't want that agent inside your actual user session.
