# mqttx

> **Cross-platform MQTT 5.0 client + CLI** for connecting,
> publishing, subscribing, benchmarking, and scripting against
> any MQTT broker (EMQX, Mosquitto, HiveMQ, AWS IoT Core, Azure
> IoT Hub-style brokers, etc.). Pinned to **v1.13.0** (released
> 2026-01-23,
> [LICENSE](https://github.com/emqx/MQTTX/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/emqx/MQTTX>

## TL;DR

`mqttx` (the CLI half of the MQTTX project, distinct from the
desktop GUI) is the tool you reach for when you need to *poke* an
MQTT broker from a terminal or a script — connect with TLS, mint
a client ID, publish a payload, subscribe to a wildcard topic, or
hammer the broker with thousands of concurrent connections to
benchmark it. It speaks MQTT 3.1, 3.1.1, and 5.0 (including
properties, user properties, request/response, shared
subscriptions), supports WebSocket transports, client
certificates for mTLS, and JSON / Base64 / hex payload framing.
A first-class `bench` subcommand turns the same binary into a
load generator (connect / pub / sub bench), so you do not need a
separate tool for "does this broker survive 10k clients?".

## Install

```bash
# npm (works anywhere Node 18+ runs)
npm install -g mqttx

# Homebrew (macOS / Linux)
brew install emqx/mqttx/mqttx-cli

# Static binary (Linux / macOS / Windows)
# https://github.com/emqx/MQTTX/releases  (mqttx-cli-*)

# verify
mqttx --version    # 1.13.0
```

## Examples

```bash
# subscribe to a wildcard topic over TLS, pretty-print JSON payloads
mqttx sub -h broker.example.com -p 8883 -l mqtts \
  -t 'sensors/+/temperature' --format json

# publish a retained message with MQTT 5 user properties
mqttx pub -h broker.example.com -p 1883 \
  -t 'config/room-12/setpoint' -m '{"target":21.5}' \
  --retain -V 5 -up "source=ops" -up "actor=$USER"

# request / response (MQTT 5) — wait for a reply on a generated topic
mqttx pub -V 5 -h broker.example.com \
  -t 'rpc/door/open' -m '{"id":"door-3"}' \
  --response-topic 'rpc/door/open/reply' --waitForReply

# benchmark: 5000 concurrent subscribers on a wildcard, 30s ramp-up
mqttx bench sub -c 5000 -i 6 -h broker.example.com -t 'sensors/#'

# benchmark publish: 1000 clients, each pushing 10 msg/s for 60s
mqttx bench pub -c 1000 -I 100 -t 'load/test/{{index}}' -m 'hello' -L 60

# inspect what your client is sending — full MQTT packet trace
mqttx pub -h broker.example.com -t a -m b --debug
```

## Use when

- You need a *single* MQTT tool that covers ad-hoc pub/sub, mTLS
  smoke tests, and load benchmarking without dragging in three
  different binaries.
- You actually use MQTT 5 features (user properties,
  request/response, shared subscriptions) — `mosquitto_pub` and
  `mosquitto_sub` only partially expose them; `mqttx` exposes
  them as first-class flags.
- You ship MQTT clients across teams and want a tool whose flags
  match the GUI (MQTTX desktop) so support tickets become
  reproducible (`mqttx pub …` instead of "click the green
  button").
- You want a benchmark you can paste into a CI job: `mqttx bench`
  exits non-zero on connection / publish failures, prints a
  structured summary, and accepts `--save` to persist the run.

Skip `mqttx` for "I have one shell line in a Dockerfile that
publishes one message" — `mosquitto_pub` is 200 KB and already
on most images. `mqttx` earns its keep when MQTT is a real part
of your stack and you need a tool that grows with it.
