# mtr

Network diagnostic tool combining `traceroute` and `ping` into a single live view.

- **Repo**: https://github.com/traviscross/mtr
- **Version**: v0.95
- **License**: GPL-2.0 — `COPYING`
- **Language**: C
- **Category**: networking / diagnostics

## Why it's interesting

Continuously probes every hop on the path to a destination and shows
per-hop loss%, latency (avg/best/worst/stddev), and jitter in a single
auto-refreshing TUI. Indispensable for debugging flaky connections to
remote inference endpoints, model registries, or self-hosted gateways
where a plain `ping` to the destination tells you nothing about which
hop along the way is dropping packets.

## Install

```sh
brew install mtr
# or on Debian/Ubuntu
sudo apt install mtr-tiny
```

## Example invocation

```sh
sudo mtr --report --report-cycles 30 api.openai.com
```
