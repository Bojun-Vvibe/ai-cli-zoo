# k6

> **A scriptable load tester for HTTP, gRPC, WebSocket, and browser
> workloads — Go binary that runs JavaScript test scripts to drive
> realistic concurrent virtual users with metrics, thresholds, and
> CI-friendly exit codes.** Pinned to **v1.7.1**
> ([LICENSE.md](https://github.com/grafana/k6/blob/master/LICENSE.md),
> AGPL-3.0).

Source: <https://github.com/grafana/k6>

## TL;DR

`k6` is what you reach for when "is this endpoint fast enough?" has
to become a number, a graph, and a CI gate instead of a vibe. You
write a test as a small JavaScript file (k6 embeds a Go-based JS
runtime — `goja` — so there is no Node, no `npm install`, and no
event loop oddities), declare a load profile (constant rate, ramp,
spike), and run `k6 run script.js`. The binary spins up many
goroutine-backed "virtual users" that each execute the script's
default function, collects per-request metrics (latency
percentiles, throughput, error rate, custom counters / trends /
gauges), and prints a summary plus optional streaming output to
Prometheus / InfluxDB / Grafana Cloud / JSON. Pass/fail comes from
declarative `thresholds` (e.g. `http_req_duration: ['p(95)<500']`),
which makes it a drop-in CI step.

## Install

```bash
# Homebrew (macOS / Linux)
brew install k6

# Linux package managers
sudo gpg -k && sudo gpg --no-default-keyring \
  --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 \
  --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt update && sudo apt install k6

# Pre-built binary (releases page)
curl -sSL https://github.com/grafana/k6/releases/download/v1.7.1/k6-v1.7.1-linux-amd64.tar.gz \
  | tar -xz --strip-components=1 -C /usr/local/bin k6-v1.7.1-linux-amd64/k6

# Verify
k6 version    # k6 v1.7.1
```

## License

AGPL-3.0 — see
[LICENSE.md](https://github.com/grafana/k6/blob/master/LICENSE.md).
**Copyleft and network-aware.** Using the `k6` binary as a CLI to
load-test your own services is fine and imposes no obligations on
your code under test. Linking k6's source into another product, or
offering a modified k6 as a network service, triggers AGPL's
source-disclosure requirements. The test scripts you write are
your own JavaScript and are not derivative works of k6. Grafana
Labs also runs a hosted k6 (Grafana Cloud k6) under different
commercial terms; the OSS binary remains AGPL-3.0.

## One Concrete Example

```bash
# 1. simplest possible script — one VU, one iteration, one GET
cat > smoke.js <<'EOF'
import http from 'k6/http';
import { check } from 'k6';

export default function () {
  const r = http.get('https://api.example.com/health');
  check(r, { 'status is 200': (res) => res.status === 200 });
}
EOF
k6 run smoke.js

# 2. realistic load profile: ramp 0->50 VUs over 30s, hold 2min,
#    ramp down. Threshold turns it into a CI gate.
cat > load.js <<'EOF'
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 50 },
    { duration: '2m',  target: 50 },
    { duration: '30s', target: 0  },
  ],
  thresholds: {
    http_req_failed:   ['rate<0.01'],          // <1% errors
    http_req_duration: ['p(95)<500', 'p(99)<1500'],
  },
};

export default function () {
  const r = http.get('https://api.example.com/items');
  check(r, { 'ok': (res) => res.status === 200 });
  sleep(1);
}
EOF
k6 run load.js
# k6 exits non-zero if any threshold is breached -> CI fails

# 3. spike test: jump from 0 to 500 VUs in 10s
k6 run --vus 500 --duration 10s smoke.js

# 4. constant arrival rate (open model — RPS, not VU count):
#    100 requests/sec for 5 minutes regardless of latency
cat > rps.js <<'EOF'
import http from 'k6/http';
export const options = {
  scenarios: {
    contacts: {
      executor: 'constant-arrival-rate',
      rate: 100, timeUnit: '1s',
      duration: '5m', preAllocatedVUs: 50, maxVUs: 200,
    },
  },
};
export default function () { http.get('https://api.example.com/items'); }
EOF
k6 run rps.js

# 5. WebSocket load test — sustained 100 concurrent connections
cat > ws.js <<'EOF'
import ws from 'k6/ws';
import { check } from 'k6';
export const options = { vus: 100, duration: '2m' };
export default function () {
  ws.connect('wss://api.example.com/feed', null, (socket) => {
    socket.on('open', () => socket.send(JSON.stringify({op:'sub'})));
    socket.on('message', (msg) => check(msg, { 'has data': (m) => m.length > 0 }));
    socket.setTimeout(() => socket.close(), 30000);
  });
}
EOF
k6 run ws.js

# 6. stream metrics to Prometheus remote-write while running
k6 run -o experimental-prometheus-rw load.js

# 7. emit machine-readable summary for downstream tooling
k6 run --summary-export=summary.json load.js
jq '.metrics.http_req_duration.values["p(95)"]' summary.json
```

## Niche It Fills

**Express load tests as code, with a real scripting language and
realistic concurrency, then fail CI on threshold breach instead of
eyeballing a histogram.** Older tools (`ab`, `wrk`, `siege`) ship a
fixed request shape and read out averages; that is fine for "hit one
URL N times" but useless for "log in, fetch a token, paginate
through 5 pages, post a comment, then disconnect — like a real
user." `k6` lets you express that flow in JS, parameterize it with
test data, share state across iterations, assert on responses, and
emit pass/fail. It is the modern, scriptable, CI-native load tester.

## Why use it

Three things `k6` does that the alternatives do not:

1. **Scenarios with explicit executors — closed model and open
   model.** `k6` distinguishes "N concurrent virtual users each
   looping" (closed model, classic VU count, what `wrk`/`ab`
   approximate) from "N requests per second injected regardless
   of how fast the server responds" (open model, `constant-
   arrival-rate` / `ramping-arrival-rate`). The open model is
   what you actually want for SLO testing — a slow server under
   `wrk` quietly reduces RPS, hiding the regression. `k6` keeps
   pushing and surfaces the queue depth and latency degradation
   honestly.
2. **Thresholds make it a CI gate, not a one-off measurement.**
   `thresholds` declares pass/fail conditions per metric
   (`http_req_duration: ['p(95)<500']`,
   `checks: ['rate>0.99']`, custom Trend / Counter metrics).
   k6 exits non-zero when any threshold is breached, so a GitHub
   Action / Jenkins / GitLab pipeline can block a release on
   "p95 latency regressed past 500 ms" with no extra glue.
3. **One binary covers HTTP/1.1, HTTP/2, WebSocket, gRPC, and
   browser (via `k6/browser`, Chromium-driven).** Most load
   tools cover one protocol; `k6` covers the realistic mix a
   modern app actually exposes. The same script can do a REST
   warmup, open a WebSocket, and call a gRPC method, sharing
   auth tokens and metrics across all of them.

For a release engineer wiring "block deploy if p95 regresses,"
the entire CI step is `k6 run --quiet --summary-export=out.json
load.js` and the exit code is the gate. No separate metrics
parser, no awk on a text report.

## Vs Already Cataloged

- **Vs [`hyperfine`](../hyperfine/):** different scope.
  `hyperfine` benchmarks a *command* (cold/warm runs of a
  binary) with statistical rigor — it is for "is `rg` faster
  than `grep` on this corpus?", not "can my API handle 500
  RPS?". `k6` benchmarks a *service* under concurrent load with
  a scripted workload. Use `hyperfine` for CLI / batch jobs,
  `k6` for network services.
- **Vs [`hurl`](../hurl/):** complementary. `hurl` writes
  declarative HTTP integration tests (one user, sequential
  assertions); `k6` runs the same kind of flows under
  concurrency. A common pattern is `hurl` for the functional
  contract test in CI on every PR, `k6` for a nightly load test
  with thresholds.
- **Vs [`grpcurl`](../grpcurl/) / [`websocat`](../websocat/):**
  those are interactive single-call debug tools. `k6`'s
  `k6/net/grpc` and `k6/ws` modules are for sustained concurrent
  load against the same protocols. Same protocol family,
  different question (debug one call vs measure many).
- **Vs [`httpstat`](../httpstat/):** orthogonal. `httpstat`
  visualizes one request's TCP/TLS/HTTP timings. `k6` aggregates
  the same timings across thousands of requests into
  percentiles, but does not break out a single waterfall. Use
  `httpstat` to figure out why one request is slow; `k6` to
  prove how many slow ones you have at p95.

## Caveats

- **AGPL-3.0 has implications if you redistribute or wrap k6.**
  Running `k6 run` against your own services from CI is
  unaffected. Embedding k6's source into a product, or offering
  a modified k6 as a hosted service, requires you to release
  your modifications under AGPL. Most teams use it as a CI
  binary and never touch this; if you are building a
  load-testing SaaS, read the license carefully.
- **JS runtime is `goja`, not Node.** No `npm install`, no
  `require('fs')`, no native modules, no async/await on
  arbitrary callbacks. You get k6's stdlib (`k6/http`, `k6/ws`,
  `k6/net/grpc`, `k6/crypto`, `k6/encoding`) plus pure-JS
  packages bundled with `k6 archive` or built via the `xk6`
  extension system. Don't expect to drop a Node script in.
- **VU concurrency is CPU-bound past a point.** A single k6
  process scales to tens of thousands of VUs on a beefy box,
  but past that you need distributed mode (Grafana Cloud k6 or
  the OSS `k6 operator` for Kubernetes). Don't try to drive
  100k VUs from a laptop and trust the numbers.
- **Browser module is Chromium-heavy.** `k6/browser` (formerly
  xk6-browser) launches real Chromium instances per VU — great
  fidelity, terrible density. Use it for a handful of VUs to
  measure end-to-end frontend perf, not for hundreds of
  concurrent browser users.
- **Default summary is human-pretty, not machine-pretty.** For
  CI dashboards use `--summary-export=summary.json` (stable
  schema) or stream to Prometheus / InfluxDB / Grafana Cloud
  with `-o`. Don't grep the terminal output.
