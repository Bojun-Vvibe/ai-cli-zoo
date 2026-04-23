# smol-developer

> Snapshot date: 2026-04. Upstream: <https://github.com/smol-ai/developer>

The original "give me a paragraph, get a whole repo" generator. Tiny,
opinionated, single-shot. Not a daily driver — a project bootstrapper.

## 1. Install footprint

- Clone the repo, `pip install -r requirements.txt`. There's no installable
  package — it's deliberately a ~200-line Python script you run in place.
- ~10 MB total.
- Python 3.10+. macOS, Linux, Windows.

## 2. License

MIT.

## 3. Models supported

OpenAI by default (`gpt-4` / `gpt-4o`). Easily monkey-patched to anything
else, but the upstream README assumes OpenAI.

## 4. MCP support

**No.** This tool predates MCP and intentionally stays minimal.

## 5. Sub-agent model

Two-pass pipeline, not really sub-agents:
1. **Plan**: one LLM call to enumerate the file tree.
2. **Generate**: one LLM call per file, in parallel, to write its contents.

There's a "shared dependencies" doc generated between passes so files
stay consistent.

## 6. Telemetry stance

**Off.** It's a script — no telemetry layer to opt out of.

## 7. Prompt-cache strategy

None. Each per-file generation is a fresh call. Costs are bounded only
by the number of files in your generated tree.

## 8. Hot keybinds

None — it's not interactive. Single command:

```bash
python main.py "a Flask app that serves a todo list with SQLite"
```

Output goes to `./generated/`. You then `cd generated && git init`.

## 9. Killer feature, weakness, when to choose

- **Killer:** the **time-to-first-tree**. From "I have an idea" to "I have
  a runnable scaffold of 15 files" in under two minutes. No setup, no
  config, no chat. The simplicity is the feature.
- **Weakness:** terrible at iteration. Once the tree exists, you're done
  with smol-developer; further edits need a different tool. Re-running
  with a new prompt regenerates from scratch.
- **Choose it when:** you're starting something from absolute zero, you
  want a vibe-coded skeleton to react to, and you'll switch to `aider` /
  `opencode` / `cline` for the actual development work after.
