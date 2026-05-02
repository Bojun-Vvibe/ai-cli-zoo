# otel-cli

> **A single Go binary that lets shell scripts and one-shot
> commands emit OpenTelemetry traces to any OTLP endpoint** —
> wrap any command in `otel-cli exec` and a span with stdout,
> stderr, exit code, duration, and `process.command_args` shows
> up in your tracing backend; run `otel-cli span` to manually
> emit a custom span from inside a shell pipeline.
> Pinned to **v0.4.5**
> ([LICENSE](https://github.com/equinix-labs/otel-cli/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/equinix-labs/otel-cli>

## TL;DR

`otel-cli` is the missing piece that lets a `bash` cron job, a
Makefile, a CI step, or a `git` post-commit hook participate in
the same distributed trace as your services. Most OTel SDKs
assume a long-lived process with an in-memory exporter queue;
shell scripts have neither, so they normally show up as gaps in
the trace. `otel-cli` solves the impedance mismatch by being a
real OTLP/gRPC + OTLP/HTTP client that starts, sends one span,
and exits — no daemon, no collector required at the host level
if your backend speaks OTLP directly. Set `OTEL_EXPORTER_OTLP_ENDPOINT`
and `TRACEPARENT` once and every wrapped command joins the
parent trace automatically via W3C Trace Context propagation.

## Install

```bash
# Homebrew (macOS / Linux)
brew install otel-cli

# Go install (any platform with Go 1.21+)
go install github.com/equinix-labs/otel-cli@v0.4.5

# Pre-built release binary
curl -LO "https://github.com/equinix-labs/otel-cli/releases/download/v0.4.5/otel-cli_0.4.5_linux_amd64.tar.gz"
tar -xzf otel-cli_0.4.5_linux_amd64.tar.gz
sudo mv otel-cli /usr/local/bin/

# .deb / .rpm / .apk
# Debian/Ubuntu: dpkg -i otel-cli_0.4.5_linux_amd64.deb
# Fedora/RHEL:   rpm -i otel-cli_0.4.5_linux_amd64.rpm
# Alpine:        apk add --allow-untrusted otel-cli_0.4.5_linux_amd64.apk

# verify
otel-cli --version    # otel-cli 0.4.5
```

The binary is a single static Go executable around 5 MB; no
runtime, no Python, no daemon. Drop it in `/usr/local/bin` on a
build agent and every shell script gains tracing.

## License

Apache-2.0 — see
[LICENSE](https://github.com/equinix-labs/otel-cli/blob/main/LICENSE).
Permissive; embedding the binary in a paid product, shipping it
inside a Docker image, or vendoring it in a build toolchain is
fine.

## One Concrete Example

```bash
# 1. point at any OTLP/gRPC backend (Jaeger, Tempo, Honeycomb,
#    Lightstep, Datadog OTLP, a local collector, etc.)
export OTEL_EXPORTER_OTLP_ENDPOINT="https://otlp.example.com:4317"
export OTEL_EXPORTER_OTLP_HEADERS="x-api-key=$OTLP_KEY"
export OTEL_SERVICE_NAME="nightly-build"

# 2. wrap a command — span auto-created, exit code preserved
otel-cli exec --name "npm install" -- npm install

# 3. nest spans by passing a parent traceparent
TP=$(otel-cli span background --name "release-pipeline" --tp-print | grep TRACEPARENT | cut -d= -f2)
export TRACEPARENT="$TP"
otel-cli exec --name "build"  -- make build
otel-cli exec --name "test"   -- make test
otel-cli exec --name "deploy" -- ./deploy.sh

# 4. emit a span manually from inside a script
otel-cli span \
  --name "db.migration.apply" \
  --service "schema-tool" \
  --kind client \
  --attrs "db.system=postgresql,db.statement=ALTER TABLE …,migration.id=2026_05_02_001" \
  --start "$(date +%s.%N)" \
  --end "$(date +%s.%N)"

# 5. propagate trace context into a child process via env
otel-cli exec --name "ansible-play" --tp-export -- \
  bash -c 'ansible-playbook -e "traceparent=$TRACEPARENT" site.yml'

# 6. error spans: exec marks span status = ERROR on non-zero exit
otel-cli exec --name "flaky-step" -- bash -c 'exit 1'   # span tagged error

# 7. send to a local OTel Collector instead of a SaaS backend
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4317"
otel-cli exec --name "smoke-test" -- curl -fsS https://api.example.com/health
```

## Niche It Fills

**OpenTelemetry tracing for shell scripts, Makefiles, CI jobs,
and one-shot CLIs that don't have an SDK in their language.**
Every OTel SDK assumes you can start a tracer, hold a span
context in process memory, and flush on shutdown — none of
which a 50-line bash script wants to do. `otel-cli` collapses
that to two commands (`exec` and `span`) and one env var
(`TRACEPARENT`), so the same trace can stitch together a Go
service, a Python worker, and the shell glue between them. It
is the standard answer to "how do I trace my CI pipeline" in
the OTel ecosystem.

## Why use it

Three things `otel-cli` does that explain why it stays in the
toolbox after teams adopt OTel SDKs in their main services:

1. **Zero-SDK tracing for the parts of your system written in
   shell.** CI pipelines, cron jobs, deployment scripts,
   `Makefile` targets, and ad-hoc `bash` glue all become first-
   class trace participants. You don't have to rewrite a build
   script in Go just to see its spans next to the service it
   builds.
2. **W3C Trace Context propagation across language boundaries
   for free.** `--tp-export` writes `TRACEPARENT` to stdout in
   `KEY=value` form; `--tp-required` refuses to emit if no
   parent context is set; `--tp-carrier` reads/writes a file so
   you can hand a parent span to a child process spawned 10
   minutes later. This is the part that's tedious to get right
   manually and easy to get wrong.
3. **Honest exec semantics.** `otel-cli exec` is a real
   `execve`-style wrapper: it forwards signals, preserves the
   child's exit code as its own, and tags the span with
   `process.command`, `process.command_args`, and the actual
   exit status. You can drop `otel-cli exec --name X --` in
   front of any command in a `Makefile` and behavior is
   identical except a span shows up.

For an LLM-CLI workflow, wrapping each agent step in
`otel-cli exec --name "step.<n>"` gives you a trace where every
tool call, every shell-out, and every retry is a span with
duration, stdout size, and exit code — the same view the SRE
team uses for the production service, applied to the agent
loop.

## Vs Already Cataloged

- **Vs [`otel-collector`](../otel-collector/):** The Collector
  is a long-running daemon that receives, processes, and
  re-exports telemetry; `otel-cli` is the upstream emitter.
  They are complements: scripts → `otel-cli` → Collector → SaaS
  backend. Pick the Collector when you need batching, retries,
  sampling, attribute scrubbing, or fan-out to multiple
  backends; pick `otel-cli` to actually generate the spans
  inside shell scripts.
- **Vs `curl` to the OTLP/HTTP endpoint directly:** Yes, you
  can `curl -X POST` a hand-crafted protobuf to `:4318/v1/traces`,
  and people do for one-off demos. `otel-cli` handles the
  protobuf encoding, gRPC plumbing, TLS, header injection,
  trace ID generation, timestamp precision (nanoseconds), and
  W3C parent context for you, in a single binary. The DIY route
  works once and breaks on the second backend.
- **Vs language-native SDKs (`opentelemetry-go`,
  `opentelemetry-python`, etc.):** SDKs are the right answer
  inside a long-lived service — they batch, they sample, they
  hook into framework middleware. They are the wrong answer
  for a 200ms shell script: SDK init alone often costs more
  than the work you're tracing, and the BatchSpanProcessor
  flush on exit is fiddly. Use SDKs in services, `otel-cli` in
  scripts.

## Caveats

- **Span = process lifetime in `exec` mode.** If your wrapped
  command spawns background work and returns, the span closes
  when the foreground process exits — the background work is
  not traced. Use `otel-cli span background` + explicit
  `--end` to model long-running async work.
- **OTLP endpoint must be reachable from the script's host.**
  `otel-cli` is not a queue. If the backend is down, the span
  is dropped (with a stderr warning) and the wrapped command
  still runs. For lossy networks, point at a local Collector
  with persistent queueing rather than a remote SaaS endpoint.
- **No metrics, no logs.** Despite the name, `otel-cli` is
  trace-only as of 0.4.x. For metrics from shell, use
  `statsd`-style emitters or push to a Pushgateway; for logs,
  use the Collector's `filelog` receiver.
- **TLS to plaintext-OTLP endpoints requires `--insecure`.**
  By default `otel-cli` assumes TLS on port 4317; set
  `--insecure` (or `OTEL_EXPORTER_OTLP_INSECURE=true`) for
  local-collector setups, or you'll get cryptic gRPC connection
  errors.
- **Release cadence is slow.** v0.4.5 has been the latest tag
  for over a year; the project is stable but not actively
  feature-developed. Pin to a known version and don't expect
  rapid bugfixes — but the existing surface is well-tested in
  production CI pipelines.
