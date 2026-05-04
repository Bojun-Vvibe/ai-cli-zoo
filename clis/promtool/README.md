# promtool

> **The official Prometheus CLI for offline validation, query
> debugging, rule unit-testing, and TSDB inspection.** Ships
> inside the same release tarball as the `prometheus` server
> binary; lives at `prometheus/cmd/promtool` upstream. Pinned to
> **v3.11.3** (released 2026-04-27, SPDX: `Apache-2.0`,
> [LICENSE](https://github.com/prometheus/prometheus/blob/main/LICENSE)).

Source: <https://github.com/prometheus/prometheus> (`cmd/promtool`)

## Repo

- URL: <https://github.com/prometheus/prometheus>
- Subpath: `cmd/promtool`
- Owner/org: prometheus (CNCF graduated project)
- License file: [LICENSE](https://github.com/prometheus/prometheus/blob/main/LICENSE)

## Version

`v3.11.3` — released 2026-04-27, distributed as part of
`prometheus-3.11.3.<os>-<arch>.tar.gz` from the
[prometheus/prometheus releases](https://github.com/prometheus/prometheus/releases/tag/v3.11.3)
page. Verify with `promtool --version`. The Prometheus 3.x line
introduced UTF-8 metric / label names by default, the
remote-write 2.0 protocol, and several PromQL changes (range
selector behavior, `info` function); promtool tracks those server
changes in lock-step.

## License

**Apache-2.0** — OSI-approved, permissive. Safe to embed in CI
runner images, ship as part of an internal observability
platform installer, or vendor into a deployment pipeline.
Same license as the upstream prometheus server.

## What it does

`promtool` is the offline / dev-time companion to a running
Prometheus server. The subcommands cluster into four jobs:

1. **Configuration validation.**
   - `promtool check config prometheus.yml` — full schema +
     semantic validation of the server config including
     scrape-job structure, relabel rules, alertmanager block,
     remote-write / remote-read endpoints, storage settings.
   - `promtool check rules rules/*.yml` — validates recording-
     and alerting-rule files.
   - `promtool check service-discovery prometheus.yml job_name`
     — runs an SD config and prints the targets it would
     discover, without starting the server.
   - `promtool check web-config web.yml` — validates TLS / basic
     auth files for the HTTP server.
2. **Rule and alert unit tests.**
   - `promtool test rules tests/*.yml` — executes a YAML-defined
     test suite that loads synthetic series, advances simulated
     time, and asserts on the values of recording rules and on
     the firing state / labels / annotations of alerts. The
     correct way to land an alerting rule change with confidence.
3. **PromQL debugging.**
   - `promtool query instant <url> '<expr>'` — runs an instant
     query against a server.
   - `promtool query range <url> '<expr>' --start --end --step`
     — range query, output as a table or JSON.
   - `promtool query series <url> --match=...` — list series
     matching a selector.
   - `promtool query labels <url> <label>` — list label values.
   - `promtool query analyze <url> '<expr>'` — analyze the
     query plan (3.x).
4. **TSDB inspection and maintenance.**
   - `promtool tsdb analyze <data-dir>` — cardinality and
     compaction stats for a stopped server's data directory.
   - `promtool tsdb dump <data-dir> --match=...` — dump samples
     to a text format for offline analysis.
   - `promtool tsdb create-blocks-from openmetrics <file>
     <data-dir>` — backfill historical samples into a server's
     storage from an OpenMetrics dump.
   - `promtool debug pprof <url>` / `metrics <url>` / `all
     <url>` — collect a server diagnostic bundle.

`promtool push metrics <url>` pushes OpenMetrics text to a
remote-write endpoint — useful for ad-hoc backfill from scripts.

## When to use

- **You ship Prometheus configuration in git.** Wire `promtool
  check config` and `promtool check rules` into CI so a typo
  in a relabel rule or a `for: 5m` that should be `for: 5h`
  fails the PR, not the on-call rotation.
- **You write alerting rules.** Treat `promtool test rules`
  the same way you treat `pytest` — every alert change ships
  with a YAML test asserting the alert fires (or does not
  fire) on the synthetic input series.
- **You triage "is this query right?" tickets.** `promtool
  query instant` / `range` against a staging server is the
  fastest way to iterate on a PromQL expression without
  pasting it into the UI repeatedly.
- **You need to backfill historical metrics.** `promtool tsdb
  create-blocks-from openmetrics` is the supported path.
- **You need a cardinality audit of a misbehaving instance.**
  `promtool tsdb analyze` on a stopped server's `data/` reports
  the worst-offender label values without standing up a
  separate analysis stack.

## When NOT to use

- **You operate Mimir / Cortex / Thanos** as the long-term
  storage layer. promtool's TSDB subcommands target a single
  server's local TSDB and won't talk to the distributed
  variants — those have their own CLIs (`mimirtool`,
  `cortextool`, `thanosctl`).
- **You want a metric-pipeline IDE.** promtool is a CLI; for
  a query-builder UI, use the Prometheus web UI, Grafana
  Explore, or [`parca`](../parca/) for profiles.
- **You want alert-fatigue scoring or routing analysis.**
  promtool only validates rule syntax / behavior; the
  *quality* of the alerting tree is a separate problem.

## Alternatives in this catalog

- [`vegeta`](../vegeta/) / [`k6`](../k6/) / [`oha`](../oha/)
  — load generators that *produce* metrics promtool then
  validates queries against. Orthogonal, complementary.
- [`grafana-cli`](../grafana-cli/) — only mentioned for
  contrast; promtool is server-side, the Grafana CLI is
  dashboard / plugin lifecycle. They sit on opposite ends of
  the same observability stack.
- [`logfire`](../logfire/) / [`otel-cli`](../otel-cli/) —
  observability tooling on the *traces* / *logs* axis;
  promtool is metrics-only.
- [`opencost`](../opencost/) — consumes Prometheus metrics
  to compute Kubernetes cost; the rules promtool validates
  end up powering opencost's joins.

## AI-native angle

promtool is not an LLM tool, but its deterministic outputs
make it the right verification step inside an
agent-generated-config loop:

- **Lint generated configs deterministically.** When an
  agent ([`aider`](../aider/), [`opencode`](../opencode/),
  [`claude-code`](../claude-code/), [`codex`](../codex/))
  edits `prometheus.yml` or a recording rule, `promtool
  check config` / `check rules` returns a structured pass /
  fail with line numbers — a ground-truth signal more
  reliable than the LLM re-reading its own output.
- **Unit-test generated alerts.** An agent that proposes a
  new alert rule should also generate the matching `promtool
  test rules` YAML and the loop should reject the PR if the
  test does not pass — closing the loop on "the alert
  *would have* fired on this synthetic incident."
- **Cardinality budget enforcement.** `promtool tsdb
  analyze` output (top label values, total series count) is
  a JSON-friendly artifact an agent can parse and gate on
  before approving a relabel-rule change that might explode
  series cardinality.
- **Query iteration without UI.** `promtool query instant`
  lets an agent iterate on PromQL inside a tool-use loop
  without screenshotting a dashboard.

## Caveats

- **Bundled with the server.** There is no standalone
  promtool release — install the prometheus tarball and
  symlink `promtool` from it, or install via a package
  manager that splits it (`brew install prometheus`
  installs both binaries).
- **Version skew matters.** promtool from Prometheus 3.x
  knows about UTF-8 names and remote-write 2.0; using it
  against a 2.x server (or vice versa) will silently miss
  some checks. Pin the version in CI to match the server.
- **Rule-test YAML is its own micro-language.** Read the
  upstream
  [unit testing for rules](https://prometheus.io/docs/prometheus/latest/configuration/unit_testing_rules/)
  doc — `interval`, `input_series`, `eval_time`,
  `exp_samples`, `alert_rule_test`, `exp_alerts` — before
  rolling your first test, otherwise the failure messages
  feel cryptic.
- **`promtool check rules` only validates syntax + structure,
  not behavior.** A rule that says `expr: 1` will pass
  `check`; only `test rules` proves it does what you think.

## Concrete example

`prometheus.yml`, `rules.yml`, and a unit test:

```yaml
# prometheus.yml
rule_files:
  - rules.yml
scrape_configs:
  - job_name: app
    static_configs:
      - targets: ['app:8080']
```

```yaml
# rules.yml
groups:
  - name: app.alerts
    rules:
      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[5m]))
              / sum(rate(http_requests_total[5m])) > 0.05
        for: 10m
        labels:
          severity: page
        annotations:
          summary: ">5% 5xx for 10m"
```

```yaml
# rules_test.yml
rule_files:
  - rules.yml
evaluation_interval: 1m
tests:
  - interval: 1m
    input_series:
      - series: 'http_requests_total{status="200"}'
        values: '100x60'
      - series: 'http_requests_total{status="500"}'
        values: '10x60'
    alert_rule_test:
      - eval_time: 11m
        alertname: HighErrorRate
        exp_alerts:
          - exp_labels:
              severity: page
            exp_annotations:
              summary: ">5% 5xx for 10m"
```

```sh
promtool check config prometheus.yml
promtool check rules rules.yml
promtool test rules rules_test.yml
```

Wire those three commands into CI and a broken rule cannot
land on the main branch.
