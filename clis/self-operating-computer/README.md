# self-operating-computer

> Snapshot date: 2026-04. Upstream: <https://github.com/OthersideAI/self-operating-computer>
> Latest release: **v1.5.8** (2025-02-28). License: **MIT** ([`LICENSE`](https://github.com/OthersideAI/self-operating-computer/blob/main/LICENSE)).

A computer-use agent that drives the host desktop directly via screenshots
and pyautogui, instead of through a browser sandbox or RPA framework.
The CLI is `operate`; you give it a goal in English, it loops
screenshot → vision-LLM decides next action → mouse/keyboard event → repeat
until the goal is reached or you `Ctrl-C`.

## 1. Install footprint

- `pip install self-operating-computer` (~40 MB; pulls `pyautogui`, `pillow`,
  `mss`, `prompt_toolkit`, `openai`, `google-generativeai`, `anthropic`,
  `ultralytics` for the YOLO grounding model).
- Python 3.10+. Console script: `operate`.
- macOS needs Accessibility + Screen Recording permissions for the
  python interpreter; Linux needs an X server; Windows works out of the box.

## 2. License

MIT (file: `LICENSE`, SPDX `MIT`). No patent grant. Safe to embed.

## 3. Models supported

- `gpt-4o` / `gpt-4-with-ocr` / `gpt-4-with-som` (Set-of-Mark grounding)
- `claude-3` (vision)
- `gemini-pro-vision`
- `qwen-vl`
- `llava` (Ollama-served)
- Any OpenAI-compatible vision endpoint via `--model` + env override.

The OCR and SoM variants exist because raw GPT-4V is bad at pixel-precise
clicks; SoM overlays numbered boxes on the screenshot first, then asks the
model to pick a box number.

## 4. MCP

No. The tool surface is fixed: click, double-click, type, hotkey, scroll,
done, search, screenshot. No external tool plug-in path.

## 5. Agent shape

Single-agent perception-action loop. Each turn captures the full screen,
sends it (and the goal + previous-actions log) to the model, parses a
single action JSON, executes it via `pyautogui`, sleeps briefly, repeats.

## 6. Telemetry default

Off. No analytics. Prompts go only to the model provider you configured.

## 7. Killer feature

`operate -m gpt-4-with-som` — the Set-of-Mark visual prompting variant.
Massively better click accuracy than vanilla GPT-4V on dense UIs because
the model picks a labelled region instead of guessing pixel coordinates.

## 8. Primary use case

Quick automation of a desktop task that has no API: clicking through a
legacy installer, scraping a Win32-only app, demoing what a vision agent
*can* and *cannot* do today. Not a production RPA replacement — there is
no error recovery or visual diffing, and a single misclick can derail
the whole run.

## 9. Install command

```sh
pip install self-operating-computer
operate                       # default GPT-4o
operate -m gpt-4-with-som     # Set-of-Mark grounding
operate -m claude-3 --voice   # Claude + voice goal
```

> ⚠️ Grants the model live mouse + keyboard control of your machine. Run
> in a VM or fresh user account for anything beyond demos.
