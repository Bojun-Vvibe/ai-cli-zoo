# gitleaks

> **A secret scanner for git repositories** — walks the
> working tree (`gitleaks dir`), the full commit history
> (`gitleaks git`), or staged changes (`gitleaks protect
> --staged`) looking for high-entropy strings, known token
> shapes (~150 built-in rules: AWS / GCP / GitHub / Slack /
> Stripe / private keys / JWTs / DB URLs / etc.), and
> custom regex rules from a TOML config; emits SARIF / JSON
> / CSV reports that CI gates and PR-comment bots consume,
> and ships first-class `pre-commit` and GitHub Action
> integrations. Pinned to **8.30.1** (commit
> `8863af47d64c3681422523e36837957c74d4af4b`,
> [LICENSE](https://github.com/gitleaks/gitleaks/blob/master/LICENSE),
> MIT).

Source: <https://github.com/gitleaks/gitleaks>

## TL;DR

`gitleaks` is the standard "did we just commit a key?"
gate. The rule set ships with ~150 detectors covering the
shapes that actually leak — `AKIA…` AWS access keys,
`AIza…` Google API keys, `ghp_…` / `github_pat_…` GitHub
tokens, `xoxb-…` / `xoxp-…` Slack tokens, `sk_live_…`
Stripe keys, RSA / OpenSSH / PGP private-key armor blocks,
`postgres://user:pass@host/db` URLs, generic
high-entropy-string heuristics for unknown keys — each rule
a regex with optional Shannon-entropy threshold and an
allowlist for known-safe matches (test fixtures, example
docs). Three operating modes: **`gitleaks git <repo>`**
walks the entire commit graph (covers the
"oops-then-amend" leak where the secret never landed in
HEAD but is still recoverable from a dangling object);
**`gitleaks dir <path>`** scans the working tree without
caring about git history (use this on CI checkouts that
might be shallow); **`gitleaks protect --staged`** runs as
a `pre-commit` hook and refuses commits that introduce a
new finding (does *not* re-flag pre-existing leaks — that
is the job of the history scan, run separately and
remediated by rotating the key + rewriting history). Output
ships in SARIF (so GitHub Code Scanning / DefectDojo /
Polaris ingest it natively), JSON (for `jq`-driven custom
gates), CSV (for spreadsheet humans), and a colorized
terminal default. Configuration lives in
`.gitleaks.toml`: extend the built-in rules, add
project-specific patterns, allowlist false positives by
regex / commit / fingerprint / file path. Single static
Go binary; no runtime, no agent, no cloud round-trip — the
scanner runs entirely on the developer's machine or the CI
runner.

## When to choose

- **You need a `pre-commit` gate that catches secrets
  before they hit `origin`** — `gitleaks protect --staged
  --redact` in `.pre-commit-config.yaml` runs in
  milliseconds against the staged diff and refuses the
  commit if a finding lands. Cheaper and more accurate
  than `git-secrets` (the AWS-Labs predecessor) because
  the rule set is broader and the entropy heuristics have
  fewer false positives on UUIDs / hashes / JWTs.
- **You inherited a repo of unknown leak history** —
  `gitleaks git --report-format sarif --report-path
  leaks.sarif .` against the full clone gives you the
  prioritized backlog: every secret ever committed,
  whether or not it is still in HEAD, with commit /
  author / file / line / rule-id / fingerprint. Feed it
  to `gh code-scanning sarif upload` (works on any GitHub
  repo) for a real bug-tracker UI on top.
- **You want CI to fail PRs that introduce new
  secrets** — the official
  `gitleaks/gitleaks-action@v2` reads the PR diff range,
  runs `protect`-style scoped scanning, and posts inline
  PR comments with the rule that fired and the redacted
  match. Free for public repos; license-key flow for
  private (the Action itself, the binary remains MIT).
- **Your secret shapes are not in the default rule set**
  — internal company tokens, custom OAuth-client formats,
  signed-URL shapes. Add a `[[rules]]` block to
  `.gitleaks.toml` with `regex` / `entropy` /
  `keywords` / `path` filters; the rule joins the built-in
  ~150 in the same scan. Allowlist test fixtures with
  `[allowlist] paths = ["^tests/fixtures/"]` so the
  honeypot key in your unit tests does not flap the gate.
- **You want a single static binary with no runtime**
  — `brew install gitleaks`, `go install
  github.com/gitleaks/gitleaks/v8@latest`, or grab the
  GitHub release. Drop it into a CI runner image, drop
  it into a rescue USB. No Python, no Node, no `git`
  required for `dir` mode (only for `git` mode).

## Comparable CLIs already in zoo

- **Vs [`trufflehog`](https://github.com/trufflesecurity/trufflehog)**
  (not in zoo): the other dominant scanner. `trufflehog`'s
  killer feature is *live verification* — it takes a
  candidate AWS key and actually calls `sts:GetCallerIdentity`
  to confirm it works, drastically cutting false positives
  but adding a network round-trip and ambient creds risk on
  the scanner. `gitleaks` is regex / entropy only and
  faster offline; pick `gitleaks` for `pre-commit` gates and
  air-gapped CI, `trufflehog` for "is this leaked key
  still live right now?" forensics.
- **Vs [`repomix`](../repomix/):** `repomix` packs a repo
  into an LLM-shaped prompt and runs Secretlint on the way
  out the door — the secret scan is a *side effect* of the
  packing pipeline, scoped to "do not paste a key into the
  model context". `gitleaks` is a *primary* secret scanner
  with a deeper rule set, history-aware mode, SARIF /
  GitHub Code Scanning integration, and `pre-commit`
  hooks. They compose: `gitleaks` on commit and CI;
  `repomix` (with its built-in Secretlint pass) at the
  LLM-export boundary.
- **Vs [`gitui`](../gitui/) / [`lazygit`](../lazygit/):**
  those are git TUI clients — they help you stage / commit
  / rebase. `gitleaks` is a scanner that *runs against*
  the commits those clients produce. Wire `gitleaks
  protect --staged` as a `pre-commit` hook and both TUIs
  will refuse to finalize a commit that introduces a new
  finding (the hook surface is OS-level and tool-agnostic).
- **Vs OS-level secret managers (1Password CLI, Bitwarden
  CLI, AWS Secrets Manager):** those *prevent* secrets
  from being in the repo by injecting them at runtime.
  `gitleaks` is the *defense in depth* layer for when a
  developer forgot to use the manager. Both belong in a
  mature setup.

## Caveats

- **A `gitleaks git` finding is not automatically a
  rotation event** — many findings are test fixtures,
  example values from docs, or already-rotated keys still
  visible in old commits. Triage each finding before
  rotating; allowlist the false positives in
  `.gitleaks.toml` so the next scan stays green.
- **A finding in git history is not removed by deleting
  the file in HEAD.** The secret remains recoverable from
  the object database until you rewrite history (`git
  filter-repo --replace-text`, `bfg-repo-cleaner`) and
  force-push *and* every collaborator re-clones *and* the
  hosting provider garbage-collects the dangling object
  *and* you have rotated the actual credential at the
  upstream provider. Treat history rewrite as last
  resort; rotation is mandatory regardless. Commit the
  remediation in a SECURITY incident note.
- **Entropy heuristics produce false positives on UUIDs,
  long hashes, base64-encoded test data, and minified
  bundles.** The built-in rule set is well-tuned but real
  repos have surprises — budget time for an initial
  allowlist sweep on the first run, then the steady-state
  noise is low.
- **`gitleaks protect --staged` only sees the staged diff.**
  A secret committed in a *previous* commit still passes
  this gate; that is by design (it is a forward-only
  guard, not a history scrubber). Pair with a periodic
  full-history `gitleaks git` run in CI on the default
  branch.
- **The license is MIT for the scanner binary** but the
  official `gitleaks-action` for GitHub *requires a
  commercial license key for private repositories* (free
  for public). The CLI itself is unencumbered; you can
  always invoke it from a hand-written CI step (`run:
  gitleaks git --report-format sarif --report-path
  leaks.sarif .`) without the Action.
