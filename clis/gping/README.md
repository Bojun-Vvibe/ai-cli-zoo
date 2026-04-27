# gping

> **Ping, but with a graph** — a single Rust binary that runs
> `ping` against one or more hosts (or `--cmd` runs an arbitrary
> command repeatedly) and renders the round-trip latencies as a
> live ASCII / Unicode line chart in your terminal, with min /
> max / avg / jitter / packet-loss in a sidebar. Pinned to
> **v1.20.1** (commit
> `ee938d1eeb064e555c10638868e7901c87901f82`,
> [LICENSE](https://github.com/orf/gping/blob/master/LICENSE),
> MIT).

Source: <https://github.com/orf/gping>

## TL;DR

`gping 1.1.1.1 8.8.8.8 cloudflare.com` opens a single chart
overlaying the latency curves for all three targets, colour-coded,
auto-scaled, with a moving 60-second window. You see jitter
patterns and dropped packets visually instead of scrolling
through `ping`'s line-per-reply output. There's also a
`gping --cmd ./flaky-thing.sh` mode that times an arbitrary
command's wall-clock duration as the y-axis — ad-hoc latency
profiling without writing a benchmark harness.

## Install

```bash
# Homebrew (macOS / Linux)
brew install gping

# Linux package managers
# Arch:    pacman -S gping
# Fedora:  dnf install gping
# Nix:     nix-env -iA nixpkgs.gping
# Alpine:  apk add gping              (community repo)

# Cargo (any OS with a Rust toolchain)
cargo install --locked gping

# from a release archive (any OS)
curl -Lo gping.tar.gz "https://github.com/orf/gping/releases/download/gping-v1.20.1/gping-v1.20.1-x86_64-apple-darwin.tar.gz"
tar xf gping.tar.gz
sudo install gping /usr/local/bin/

# Windows
# scoop install gping
# choco install gping

# verify
gping --version    # gping 1.20.1
```

No config file, no daemon — single binary; on Linux it uses
`/bin/ping` under the hood (no raw socket / setuid required).

## License

MIT — see
[LICENSE](https://github.com/orf/gping/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. simplest: graph latency to a single host
gping 1.1.1.1

# 2. compare multiple targets on one chart (overlaid lines)
gping 1.1.1.1 8.8.8.8 9.9.9.9

# 3. resolve a hostname every probe (catch DNS / Anycast flips)
gping --resolve-every 1 cloudflare.com

# 4. force IPv4 / IPv6 for an A+AAAA host
gping -4 example.com
gping -6 example.com

# 5. tighter probe interval for short-lived investigation
gping --interval 0.2 my-router.lan

# 6. measure an arbitrary command's wall-clock duration
gping --cmd "curl -s -o /dev/null https://api.example.com/health"

# 7. compare two commands as overlaid lines
gping --cmd "psql -c 'select 1'" --cmd "psql -c 'select count(*) from big_table'"

# 8. control buffer length (default ~100 samples shown)
gping --buffer 600 1.1.1.1     # 10 minutes at 1s/probe

# 9. pure-ASCII chart for terminals without box-drawing
gping --simple-graphics 1.1.1.1

# 10. write a CSV alongside the live graph for later analysis
gping 1.1.1.1 --watch-interval 1 > /tmp/ping.csv
```

## Niche It Fills

**`ping`'s line-per-reply output makes it impossible to see
patterns over time.** Is the spike periodic? Is loss bursty or
uniform? Did the average shift after you switched VPNs? Reading
those answers out of `64 bytes from 1.1.1.1: ttl=57 time=12.3 ms`
lines requires either eyeballing thousands of rows or piping into
`awk` and a plotter. `gping` collapses that loop: the chart *is*
the analysis. It's the right tool for the 90 seconds between "is
the network weird?" and "yes, the spike is every 30 s, and only
on the VPN".

## Why use it

Three things `gping` does that the obvious alternatives do not:

1. **Multi-target overlay on one chart.** Comparing latency to
   three resolvers in `ping` requires three terminals; in `gping`
   it's one positional arg per target with auto-coloured lines.
   Convergence / divergence between targets is visible at a
   glance.
2. **`--cmd` mode = ad-hoc command timing chart.** Anything that
   exits 0/non-zero can become a timing series:
   `gping --cmd ./deploy-smoke-test.sh` graphs how long each run
   of your script takes, with the same min/max/avg sidebar — a
   30-second alternative to wiring up a real APM or running
   [`hyperfine`](../hyperfine/) in a loop.
3. **No raw socket on Linux.** Calls `/bin/ping` as a subprocess
   and parses its output, so `gping` doesn't need `CAP_NET_RAW`
   or setuid; it works the same as a regular user as it does
   under root, and inside unprivileged containers.

For agent / LLM workflows where the model is debugging a flaky
network or a slow endpoint, `gping --cmd '...'` produces
human-readable evidence you can screenshot or paste, without
the agent having to invent its own timing harness.

## Vs Already Cataloged

- **Vs [`hyperfine`](../hyperfine/):** complementary —
  `hyperfine` runs N warmups + M timed runs of a command and
  produces statistical output (mean, stddev, outlier detection)
  for benchmarking. `gping --cmd` is a live ongoing chart with
  no fixed run count, intended for "watch this over time" not
  "give me a number to put in a PR description". Use `hyperfine`
  to publish a benchmark; use `gping --cmd` to investigate a
  symptom.
- **Vs [`bottom`](../bottom/) (`btm`):** `bottom`'s network
  panel shows aggregate throughput, not per-host latency.
  `gping` is the right tool for "latency to *this* host"; pair
  it with `bottom` for "throughput on *this* interface".
- **Vs `mtr` / traceroute:** `mtr` shows per-hop loss/latency
  along the path (route diagnosis); `gping` shows endpoint
  latency over time (symptom diagnosis). Run `mtr` once to
  find which hop is bad; run `gping` continuously to confirm
  the symptom is gone after a fix.

## Caveats

- **Latency includes ping's own scheduling jitter.** Sub-millisecond
  spikes on a LAN may be the kernel scheduling `ping`, not the
  network. For sub-ms work, set `--interval 0.5` or higher and
  trust the trend line, not individual points.
- **`--cmd` runs the command in your shell with no timeout.**
  A wedged `curl` keeps the whole sample slot occupied;
  combine with `timeout 5 curl ...` if hangs are possible.
- **Buffer is in-memory only.** The default ~100-sample window
  scrolls; once a point falls off the left edge it's gone
  unless you redirected stdout to a CSV. For long captures
  (hours), redirect the CSV and graph it separately; the TUI
  is for live observation, not historical replay.
- **Windows / WSL ping flag parsing differs.** `gping` shells
  out to whichever `ping` is on `PATH`, so on Windows it parses
  the Microsoft `ping` output and on WSL it parses iputils
  `ping`; behaviour around `-c` / `-n` and DNS resolution can
  differ subtly. Prefer running `gping` from the same OS as the
  network you're diagnosing.
- **Not a packet-capture tool.** It can tell you "the link is
  bad", not "this TCP retransmit was the problem". For wire-level
  detail reach for `tshark` / `tcpdump`; `gping` is the screen
  you stare at to know whether to bother.
