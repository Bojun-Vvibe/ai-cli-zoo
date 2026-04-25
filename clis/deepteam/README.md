# deepteam

> Snapshot date: 2026-04. Upstream: <https://github.com/confident-ai/deepteam>

"**The open-source LLM red-teaming framework.**" Deepteam is the
adversarial-evaluation sibling of `deepeval`: instead of asserting
that a chat / RAG / agent system behaves *correctly* on a known
test set, it generates and runs *attacks* against the system and
asserts that the system *resists* them. 50+ pre-built
vulnerabilities (prompt injection, jailbreaks, PII leakage,
toxicity, bias, hallucination, excessive agency, RBAC bypass,
unauthorized tool use, BFLA / BOLA, …) are paired with attack
*enhancements* (linear / tree-of-attacks jailbreak, multi-turn
crescendo, ROT-13 / Base64 / leetspeak obfuscation, prompt
probing) and an LLM-as-judge metric per vulnerability returns
pass/fail with a structured reason. Outputs feed straight into
the OWASP LLM Top 10 / NIST AI RMF compliance shape.

## 1. Install footprint

- `pip install deepteam` (pulls `deepeval`, `pydantic`, `openai`,
  judge-model defaults).
- Python ≥ 3.10. macOS, Linux, Windows. No daemon, no DB; results
  land as JSON / CSV / a `deepteam` HTML report.
- Optional: `confident-ai` cloud login (`deepeval login`) for
  hosted run history and red-team comparison dashboards — fully
  optional, library mode never phones home.

## 2. Repo + version + license

- Repo: <https://github.com/confident-ai/deepteam>
- Latest release: **v1.0.4** (2025-11-12)
- License: **Apache-2.0** —
  <https://github.com/confident-ai/deepteam/blob/main/LICENSE.md>
- HEAD SHA: `bf8aa8d3976172defe3157be92ea92d97bcea04b`
- Default branch: `main`
- Language: Python

## 3. Models supported

Attacker, judge, and target models are all decoupled. Attacker /
judge default to OpenAI but accept any `deepeval` `DeepEvalBaseLLM`
subclass — OpenAI, Anthropic, Gemini, Bedrock, Azure, Mistral,
Cohere, Ollama, vLLM, LiteLLM-routed, custom HTTP. The *target*
is any callable: `def model_callback(prompt: str) -> str` —
wrap your chat endpoint, your LangChain chain, your CrewAI crew,
your agent runner, or a remote HTTP API. Multi-modal attacks
(image + text) supported when the target accepts vision input.

## 4. Simple usage

```python
from deepteam import red_team
from deepteam.vulnerabilities import Bias, Toxicity, PromptLeakage, PIILeakage
from deepteam.attacks.single_turn import PromptInjection, Roleplay
from deepteam.attacks.multi_turn import LinearJailbreaking, TreeJailbreaking

def model_callback(prompt: str) -> str:
    # any callable: your chat endpoint, agent runner, RAG pipeline
    return my_chat_app(prompt)

risks = red_team(
    model_callback=model_callback,
    vulnerabilities=[
        Bias(types=["race", "gender", "religion"]),
        Toxicity(types=["insults", "threats"]),
        PromptLeakage(types=["secrets and credentials", "instructions"]),
        PIILeakage(types=["api keys", "session tokens"]),
    ],
    attacks=[
        PromptInjection(),
        Roleplay(persona="DAN", role="jailbroken assistant"),
        LinearJailbreaking(weight=2),
        TreeJailbreaking(weight=3),
    ],
    target_purpose="A customer-support chatbot for an e-commerce site.",
    attacks_per_vulnerability_type=5,
)
risks.save("./redteam_report.json")
```

```bash
# CLI surface for CI gating
deepteam scan --target ./model_callback.py --vulnerabilities all --max-attacks 100
deepteam report ./redteam_report.json --format html > report.html
```

## 5. Why it's interesting

- **Attacks and vulnerabilities are independent matrices** —
  pick which weaknesses to probe (`Bias`, `PIILeakage`, …) and
  which attack styles to use (`LinearJailbreaking`,
  `TreeJailbreaking`, `Crescendo`, `Roleplay`, `Base64`,
  `Multilingual`, `MathProblem`, …); the framework cross-products
  them, so adding a new attack covers every vulnerability and
  vice versa with no glue.
- **Tree-of-Attacks-with-Pruning is in-tree** — multi-turn
  jailbreaks branch and prune adversarial paths against the
  target with the attacker LLM choosing which node to expand,
  not a hand-rolled loop.
- **OWASP LLM Top 10 + NIST AI RMF mapping ships with the
  vulnerabilities** — `vulnerability.owasp_id` and
  `vulnerability.nist_id` properties make compliance reports a
  one-liner instead of a manual mapping spreadsheet.
- **Library mode is fully local** — `deepteam` does not require a
  Confident AI account; the SaaS dashboard is the optional
  upsell for run history, not a hard dependency for any feature
  in the OSS package.

## 6. Caveats

- Quality of the red-team is bounded by the *attacker* model —
  use a frontier-tier judge / attacker (Claude Sonnet, GPT-4o,
  Gemini 2.x) for credible coverage; Ollama-only attacker setups
  produce shallow attacks.
- Token cost scales as `vulnerabilities × attacks × attacks_per_type
  × multi_turn_depth`; keep `attacks_per_vulnerability_type` low
  while iterating.
- LLM-as-judge metrics inherit judge bias; pair with manual spot
  review on borderline failures rather than treating the report
  as ground truth.
- Use only against systems you operate or have written
  authorization to test.
