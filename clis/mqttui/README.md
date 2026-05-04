# mqttui

> **Interactive TUI client for MQTT v3.1.1 / v5** — a single Rust
> binary (rumqttc + ratatui) that subscribes to `#`, builds a live
> tree of topics as messages arrive, lets you walk the tree with
> arrow keys, see the most recent payload + retained flag + QoS +
> last-update timestamp per topic, plot the value-over-time graph
> for numeric payloads, and publish back from the same window with
> `p` — pinned to **v0.22.1** (released 2025-04-08,
> [LICENSE](https://github.com/EdJoPaTo/mqttui/blob/v0.22.1/LICENSE),
> SPDX `GPL-3.0-or-later`).

Source: <https://github.com/EdJoPaTo/mqttui>

## TL;DR

MQTT is the dominant pub/sub protocol for IoT / home-automation /
industrial sensor fleets — Mosquitto, EMQX, HiveMQ, AWS IoT Core
all speak it. The first-line CLIs are **`mosquitto_sub` /
`mosquitto_pub`** (raw stdout / stdin, no UI, no history, no
view of the topic tree as a *shape*) and **MQTT Explorer** (Electron
GUI, ~150 MB, mouse-driven, doesn't run over SSH).

The middle gap — "I want to SSH into the broker host, run one
binary, see what's on the bus *right now* organised as a tree,
poke a value back, exit" — is exactly `mqttui`. It opens
`mqtt://broker:1883` (or `mqtts://` with TLS, or `ws://` /
`wss://` for browser-style brokers), subscribes to `#`, and
renders the tree as you'd render a filesystem: `homeassistant/` >
`sensor/` > `living_room_temperature/state` with the latest
payload `21.4` shown inline. Walk with `↑ ↓`, expand with `→`,
view the full payload + history pane on the right, hit `p` to
publish to the focused topic, `c` to clean a retained message,
`q` to exit. Numeric payloads get a sparkline over time without
configuration.

## Install

```bash
# Cargo (any host with Rust 1.74+)
cargo install mqttui

# macOS
brew install mqttui

# Arch (AUR)
yay -S mqttui

# Pre-built binaries from upstream tags (Linux x86_64 / aarch64 /
# armv7 / riscv64, macOS arm64 / x86_64, Windows x86_64 / aarch64,
# .deb + .rpm packages) live at:
#   https://github.com/EdJoPaTo/mqttui/releases/tag/v0.22.1
```

No daemon, no system services, no broker — `mqttui` is a *client*.
You need a broker reachable from the host (Mosquitto on the same
LAN, EMQX in a Docker container, HiveMQ Cloud, AWS IoT Core,
etc.).

## Common invocations

```bash
# Browse everything on a local broker
mqttui --broker mqtt://localhost:1883

# Auth + TLS to a hosted broker
mqttui --broker mqtts://hivemq.cloud:8883 \
       --username alice --password "$MQTT_PASSWORD"

# Subscribe only to a sub-tree (faster on noisy buses)
mqttui --broker mqtt://localhost --subscribe-topic 'homeassistant/#'

# One-shot publish (no TUI) — drop-in `mosquitto_pub` replacement
mqttui publish --broker mqtt://localhost \
       --topic 'homeassistant/light/desk/set' \
       --message '{"state":"ON","brightness":128}'

# One-shot subscribe (no TUI) — drop-in `mosquitto_sub`, JSON output
mqttui log --broker mqtt://localhost --subscribe-topic '#'

# Honor SSLKEYLOGFILE for Wireshark TLS decryption (v0.22 added)
SSLKEYLOGFILE=~/mqtt.keys mqttui --broker mqtts://broker:8883
```

## Why orthogonal to existing zoo

The zoo has plenty of HTTP / gRPC / WebSocket / DNS / TCP probes
(`xh`, `httpie`, `grpcurl`, `evans`, `websocat`, `dog`,
`doggo`, `mtr`, `bandwhich`) but **no MQTT client at all** —
neither a CLI publisher nor an interactive browser. MQTT is its
own protocol family with its own semantics (retained messages,
QoS 0/1/2, will messages, sticky sessions, $SYS topics) that none
of the HTTP-shaped tools speak. For anyone who runs Home
Assistant, Zigbee2MQTT, an industrial PLC stack, or a fleet of
ESP32 sensors, "open a TUI on my broker and see what's flowing"
is a daily operation that has no equivalent in the existing
catalog.

## Caveats

- GPL-3.0-or-later (vs the typical MIT / Apache-2.0 of most CLI
  zoo entries) — fine for desktop / sysadmin use, but if you
  intend to embed `mqttui`'s code into a closed-source product
  the license is the wrong shape; pick `mosquitto_sub` (BSD-3) or
  the rumqttc library directly (Apache-2.0) instead.
- Subscribing to `#` on a busy broker (thousands of messages /
  sec) is expensive — the in-memory topic tree grows unbounded
  until you exit. For large buses scope with
  `--subscribe-topic` to a sub-tree, or use `mqttui log` for
  one-shot non-retained capture.
- WebSocket transport (`ws://` / `wss://`) is supported but is
  newer than the TCP transport and has fewer test miles — for
  production debugging on browser-facing brokers (HiveMQ Cloud
  WebSocket endpoint, AWS IoT Core over WebSocket Secure with
  SigV4) verify the auth handshake works in `mqttui log` first
  before trusting the interactive view.
- Sparkline graphs assume the payload parses as a single number;
  JSON payloads (`{"temperature":21.4,"humidity":44}`) are shown
  raw and do not graph — pipe through a transformer broker or
  publish the scalars to their own topics if graphing matters.
- No MQTT v5-only features beyond what `rumqttc` implements
  (subscription identifiers, topic aliases, server redirect,
  shared subscriptions in some forms work; some advanced response
  topic / correlation data flows are not first-class in the UI
  yet).
- The auth surface is username + password + TLS client cert; for
  AWS IoT Core SigV4-over-WebSocket or Azure Event Grid / Event
  Hubs MQTT-broker auth flows that need pre-signed URLs, you
  need to mint the URL outside `mqttui` and feed it as the broker
  argument.
