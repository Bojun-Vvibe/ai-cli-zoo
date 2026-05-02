# pre-commit

> **The Python-implemented multi-language git-hook framework that
> makes a `.pre-commit-config.yaml` the single source of truth for
> every check that runs on `git commit` / `git push`, with hook
> executables fetched from upstream repos and pinned by SHA, run
> in per-hook isolated environments (Python venv / Node / Ruby /
> Rust / Go / Docker / system) and gated against the staged-file
> subset.** Pinned to **v4.4.0** (as of 2025),
> [LICENSE](https://github.com/pre-commit/pre-commit/blob/main/LICENSE),
> MIT.

Source: <https://github.com/pre-commit/pre-commit>

## TL;DR

`pre-commit` is the original cross-language hook orchestrator and
the one most third-party linters / formatters publish a
`.pre-commit-hooks.yaml` for, so adopting it gives you the widest
out-of-the-box hook catalog of any framework in this niche
(`ruff`, `black`, `isort`, `mypy`, `prettier`, `eslint`,
`shellcheck`, `shfmt`, `golangci-lint`, `terraform_fmt`,
`detect-secrets`, `gitleaks`, `markdownlint`, `yamllint`,
`hadolint`, `actionlint`, plus hundreds more — every popular
linter ships hook metadata for it). Each entry in
`.pre-commit-config.yaml` pins a hook repo by `rev:` (a tag or
commit SHA), and `pre-commit autoupdate` walks every entry to
its latest tag in one command — reproducible across machines
without checking the hook executables themselves into the repo.

## Install

```bash
# Homebrew (macOS / Linux)
brew install pre-commit

# pipx (any platform with Python 3.9+)
pipx install pre-commit

# pip (project-local virtualenv)
pip install pre-commit

# Then, in any repo:
pre-commit install            # writes .git/hooks/pre-commit
pre-commit install --hook-type pre-push --hook-type commit-msg
pre-commit run --all-files    # one-shot: run every hook against the whole tree
```

## License

[MIT](https://github.com/pre-commit/pre-commit/blob/main/LICENSE),
SPDX `MIT`.

## Niche / positioning

Pick `pre-commit` over [`lefthook`](../lefthook/) when the
project is Python-shaped or when the team wants the broadest
catalog of pre-authored hooks (every popular linter publishes a
`.pre-commit-hooks.yaml`); pick `lefthook` when the team wants
to avoid a Python runtime dependency on every contributor's
machine and wants parallel hook execution by default. Pick over
[`talisman`](../talisman/) when the goal is a *general* hook
framework — talisman is a single-purpose secret-scan hook,
pre-commit hosts secret scanners (`detect-secrets`, `gitleaks`)
as one of many concerns. Skip when the repo already standardised
on `husky` + `lint-staged` for a Node-only codebase (the Node
ecosystem's idiomatic pick), or when CI alone is the gate and
client-side hooks are not part of the policy.
