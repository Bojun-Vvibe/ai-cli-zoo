# garak

- **Repo:** https://github.com/NVIDIA/garak
- **Version:** `v0.12.0` (latest release)
- **License:** Apache-2.0 (`LICENSE`)
- **Language:** Python
- **Install:** `pip install garak` then `garak --model_type openai --model_name gpt-4o-mini --probes lmrc`

## One-line summary

An `nmap`-style **LLM vulnerability scanner**: point it at a model, pick a
probe set, and get a report of which jailbreaks, prompt injections, leaks,
and hallucination patterns it actually fell for.

## What it does

Garak is structured around four pluggable concepts:

- **Generators** — adapters for the target (OpenAI, Anthropic, HuggingFace,
  Replicate, NIM, Ollama, REST, plain `function:` Python callable, etc.).
- **Probes** — attack families: `dan` (jailbreaks), `promptinject`,
  `leakreplay` (training-data extraction), `realtoxicityprompts`,
  `xss`, `malwaregen`, `lmrc` (Language Model Risk Cards), and dozens
  more.
- **Detectors** — graders that decide whether a given response counts as a
  hit (regex, classifier, another LLM, string match).
- **Buffs** — transformations applied to prompts before they hit the
  generator (paraphrase, encode, translate) to test robustness.

A run produces a JSONL report plus an HTML summary with per-probe pass
rates, so you can diff scans across model versions and treat safety
regressions like any other CI signal.

```bash
pip install garak
garak --list_probes                          # browse the attack catalog
garak --model_type openai \
      --model_name gpt-4o-mini \
      --probes promptinject,dan,leakreplay
garak --report_prefix nightly                # for CI artifacts
```

## When to choose it

- You ship an LLM-backed product and want a **repeatable safety / jailbreak
  baseline** that runs in CI against every model swap or system-prompt
  change.
- You're evaluating providers and want **apples-to-apples robustness
  numbers** instead of vibes.
- You need to demonstrate due diligence to a security or compliance
  reviewer and want machine-readable evidence.

## When NOT to choose it

- You want a tool that *fixes* findings — garak only reports; remediation
  is on you (system prompt, guardrails, fine-tune, model swap).
- You need exhaustive coverage of a specific niche threat model not in the
  built-in probe set; expect to write custom probes/detectors.
- You're scanning a closed model with strict rate limits — full probe
  sweeps are chatty and can rack up real spend or trip abuse heuristics.
