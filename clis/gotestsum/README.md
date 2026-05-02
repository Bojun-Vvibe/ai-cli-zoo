# gotestsum

- **Repo:** https://github.com/gotestyourself/gotestsum
- **Version:** v1.13.0 (latest stable, September 2025)
- **License:** Apache-2.0 ([LICENSE](https://github.com/gotestyourself/gotestsum/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install gotestsum` · `go install gotest.tools/gotestsum@v1.13.0` · static binaries on the GitHub release page · binary name is `gotestsum`

## What it does

`gotestsum` is a drop-in front-end for `go test` that consumes the
test binary's `-json` event stream and re-renders it as a friendly,
collapsible test report instead of the raw line-per-event noise the
standard runner produces. It supports a dozen output formats
(`pkgname`, `pkgname-and-test-fails`, `testname`, `testdox`, `dots`,
`dots-v2`, `standard-quiet`, `standard-verbose`, `github-actions`,
plus a JSON pass-through for downstream tooling), writes a JUnit XML
report (`--junitfile`) and the raw JSON log (`--jsonfile`) on the
side, and surfaces a post-run summary that lists exactly which tests
failed, which were skipped, and which packages didn't compile —
without you having to scroll back through 50 000 lines of `--- PASS`.

Beyond cosmetics it bundles three behaviors the bare `go test`
toolchain doesn't have: `--rerun-fails=N` automatically re-runs
flaky tests up to N times and only reports a real failure if the
test is consistently red; `tool slowest --threshold 500ms` (operating
on the JSON log) prints the slowest tests so you can `gotestsum
--packages=./... -- -run "$(gotestsum tool slowest --threshold 1s
--print-issues)"` to attack the long pole; and the `--watch` mode
reruns only the packages whose files changed, giving a TDD-style
inner loop without an external file watcher.

## When to pick it / when not to

Pick `gotestsum` whenever you have a Go test suite larger than
"toy" — the readability win starts at maybe 50 tests and is
overwhelming by 500. The killer combination in CI is `--format
testname --junitfile report.xml --rerun-fails=2 --packages ./...`:
human-readable build logs, CI-system-readable JUnit XML for the test
report tab, and automatic retry of single-failure flakes without
re-running the whole suite.

Pick it for **local TDD** with `gotestsum --watch --format dots` —
each save shows a stream of dots and turns red exactly where
something broke, like `pytest --tb=short` for Go.

Skip it when:

- Your project's CI already standardises on a different runner
  (`go test -json | tparse`, Bazel's `go_test`, Mage targets that
  parse `-json` themselves) — `gotestsum` overlaps with all of them.
- You need **coverage gating / coverage diff** as the primary signal
  — that's [`gocov`](https://github.com/axw/gocov) +
  [`gocov-html`](https://github.com/matm/gocov-html) or
  [`go-cover-treemap`](https://github.com/nikolaydubina/go-cover-treemap),
  not a test-run formatter.
- You want a **fuzzing-first** loop — `go test -fuzz` already streams
  its own progress; `gotestsum` doesn't add much there.

## Why it matters in an AI-native workflow

LLM coding agents working on Go code live or die by how compactly
they can read test output. The default `go test ./...` log
truncates badly into a context window: thousands of `=== RUN` /
`--- PASS` lines, and the actual `FAIL` reason is buried 80 lines
above the package summary. `gotestsum --format pkgname-and-test-fails`
collapses that to one line per package plus full output for failures
only — exactly the shape an agent (or a human) needs to decide what
to fix next.

The JSON sidecar (`--jsonfile`) is the other half: feed it to a
post-processor that emits structured `{package, test, status, output,
elapsed}` rows, and an agent can answer "which tests changed status
between this run and the last green run?" without guessing. The
`--rerun-fails` behavior is also a quiet enabler — it removes the
"is this test flaky or did my patch actually break it?" debate from
the agent's reasoning surface.

## Example invocations

```bash
# Friendly local run: one line per package, full output for any failure.
gotestsum --format pkgname-and-test-fails

# TDD inner loop: rerun affected packages on save.
gotestsum --watch --format dots

# CI shape: JUnit XML for the test tab, retry single-test flakes twice,
# write the raw JSON log so post-processing can diff against last run.
gotestsum --format testname \
    --junitfile report.xml --jsonfile events.json \
    --rerun-fails=2 --packages=./... -- -race -count=1

# Pass through go test flags after `--`.
gotestsum -- -tags=integration -timeout=5m ./...

# Find the 10 slowest tests from the last run's JSON log.
gotestsum tool slowest --jsonfile events.json --threshold 500ms

# Re-run only the previously-failed tests (reads from the JSON log).
gotestsum --rerun-fails --packages=./... -- -count=1
```

## Caveats

- `--rerun-fails` re-runs are a *flaky-test mitigation*, not a fix.
  Watch the rerun count over time; if it climbs, you have a
  legitimate non-determinism bug, and silently retrying it is
  hiding tech debt. The `tool` subcommand on the JSON log lets you
  audit how often each test needed retries.
- `--watch` watches the filesystem with a polling fallback; on
  large monorepos consider scoping with `--packages ./pkg/...`
  instead of letting it walk every module.
- The JUnit output is a best-effort mapping — Go's test model
  (subtests, table tests with `t.Run`) doesn't map 1:1 to the
  JUnit `<testsuite>/<testcase>` shape. Most CI dashboards render
  it correctly, but a few corner cases (parallel subtests, skipped
  table-test entries) display oddly.
- `gotestsum` does not change Go's test selection semantics — it
  only formats the output stream. `-run`, `-skip`, `-count`,
  `-race`, `-tags`, `-cover` all behave exactly as `go test` does.
