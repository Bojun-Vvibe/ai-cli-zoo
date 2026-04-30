# termshark

> **A terminal UI for tshark / Wireshark capture
> files** — a single Go binary that wraps the
> `tshark` packet-analysis engine in a Bubble-Tea-
> style TUI, so you can browse, filter, and decode
> live or saved `.pcap` traffic from an SSH session
> without forwarding X11 or copying captures back
> to a workstation. Pinned to **v2.4.0**
> ([LICENSE](https://github.com/gcla/termshark/blob/master/LICENSE),
> MIT, © Graham Clark).

Source: <https://github.com/gcla/termshark>

## TL;DR

`termshark` is what you reach for when something
weird is happening on a remote box and you'd
normally `tcpdump -w /tmp/cap.pcap`, `scp` the
file home, and open it in Wireshark — except now
the box is a container, the file is 800 MB, and
you don't want any of that. It depends on a system
`tshark` (the Wireshark project's CLI dissector)
for the actual packet parsing — every protocol
Wireshark knows, termshark also knows, because
it's literally calling `tshark` under the hood —
and overlays a three-pane TUI that mirrors the
Wireshark layout: a packet list (top), a
protocol-tree dissection of the selected packet
(middle), and a hex/ASCII view of the bytes
(bottom). You drive it with arrow keys and
single-letter commands: `/` to apply a Wireshark
display filter (`http.request`, `tcp.port == 443
&& ip.addr == 10.0.0.5`, etc.), `n/p` to jump to
the next/previous match, `f` to follow a TCP/UDP
stream, `c` to cycle conversations, `q` to quit.
Live capture works the same way as `tshark -i`:
`termshark -i eth0` opens an interface, streams
packets into the list as they arrive, and lets you
filter on the fly. Saved-capture mode is the same
TUI but reads a `.pcap`/`.pcapng` file instead.
Output is real Wireshark filter syntax all the way
down, so muscle memory transfers in both
directions.

## Install

```bash
# Homebrew (macOS / Linux) — pulls in tshark too
brew install termshark

# Snap (Linux)
snap install termshark

# Arch Linux (community)
pacman -S termshark

# Nix
nix-env -iA nixpkgs.termshark

# Go (any platform with Go 1.20+)
go install github.com/gcla/termshark/v2/cmd/termshark@latest

# Debian / Ubuntu (universe / community repos)
sudo apt install termshark

# pre-built binaries on the releases page
# https://github.com/gcla/termshark/releases

# verify
termshark --version    # termshark 2.4.0
tshark --version       # required dependency, any recent version
```

You will also need `tshark` on the `$PATH` and
(on Linux) the usual capability bits for live
capture:

```bash
sudo setcap 'CAP_NET_RAW+eip CAP_NET_ADMIN+eip' "$(command -v dumpcap)"
```

## Basic usage

```bash
# open a saved capture
termshark -r /tmp/api-traffic.pcap

# live capture on an interface (uses tshark/dumpcap underneath)
sudo termshark -i eth0

# live capture with a BPF capture filter (kernel-level prefilter)
sudo termshark -i eth0 -f "tcp port 443"

# live capture on multiple interfaces simultaneously
sudo termshark -i eth0 -i wlan0

# pre-apply a Wireshark display filter on open
termshark -r dump.pcap -R 'http.request.method == "POST"'

# read from stdin (any tshark-compatible source)
sudo tcpdump -i eth0 -U -w - | termshark -r -

# write the capture to disk *and* show it live
sudo termshark -i eth0 -w /tmp/eth0.pcap

# convert a saved capture to PSML/PDML/JSON via the underlying tshark
termshark -r dump.pcap --pass-thru=auto -T json > dump.json
```

Inside the TUI:

| key            | action                                          |
| -------------- | ----------------------------------------------- |
| `/`            | edit display filter                             |
| `n` / `p`      | next / previous filter match                    |
| `f`            | follow TCP / UDP / TLS stream of selected pkt   |
| `c`            | conversations dialog                            |
| `tab`          | move focus between the three panes              |
| `?`            | help overlay                                    |
| `ctrl-c` / `q` | quit                                            |

## When to choose

- **You're debugging a network problem from an SSH
  session and X11 forwarding is a non-starter** —
  a misbehaving service in a Kubernetes node, a
  bastion host, an embedded device — `termshark`
  is the only fully-featured Wireshark-grade tool
  that lives entirely in the terminal.
- **The capture is too big to copy home** — multi-
  hundred-MB pcaps open quickly because the heavy
  lifting is delegated to `tshark`, and the TUI
  paginates rather than loading the whole packet
  list at once.
- **You want one tool with the same filter syntax
  on every platform** — Wireshark display filters
  (`http.request.uri matches "/api/.*"`,
  `tls.handshake.type == 1`,
  `ip.geoip.country == "DE"` if your tshark has
  GeoIP) work identically here.
- **You want to convert from a `tcpdump` muscle-
  memory user to a Wireshark muscle-memory user
  without leaving the terminal** — the TUI is a
  faithful subset of the Wireshark GUI layout.

## When NOT to choose

- **You need Wireshark's GUI-only features** —
  the I/O graphs, the expert info panel, the
  flow-graph visualisation, the SDP/RTP playback,
  the USB/Bluetooth analyser dialogs. Use
  Wireshark itself.
- **You don't have / can't install `tshark`** —
  termshark is a UI shell. Without `tshark` on
  `$PATH` it can't dissect anything. On locked-
  down boxes where you can't install Wireshark
  packages, fall back to `tcpdump -X` or
  [`tcpflow`](https://github.com/simsong/tcpflow).
- **You only need a one-line summary, not a
  dissection** — `tshark -r dump.pcap -q -z
  conv,tcp` or [`zeek`](https://zeek.org/) for
  protocol logs is faster.
- **You need to capture at multi-gigabit line
  rate** — termshark/tshark/Wireshark are all
  user-space libpcap and will drop packets under
  load. Reach for AF_PACKET ring tools like
  `tcpdump -B`, [`netsniff-ng`](http://netsniff-ng.org/),
  or DPDK-based capture.

## Why it fits the zoo

The zoo's network-tool cluster
([`bandwhich`](../bandwhich/),
[`sniffnet`](../sniffnet/),
[`havn`](../havn/),
[`gping`](../gping/),
[`trippy`](../trippy/),
[`dog`](../dog/),
[`doggo`](../doggo/),
[`xh`](../xh/)) already covers the
"summarise / probe / measure" axis. `termshark`
fills the deep-inspection slot: when summary
isn't enough and you need to look at every byte
of a flow, this is the terminal-native answer.
It pairs naturally with [`bandwhich`](../bandwhich/)
("which process is using the bandwidth?") and
[`sniffnet`](../sniffnet/) ("what does the
high-level traffic mix look like?") — when those
tell you *something* is wrong, `termshark` is
where you go to find out *what*.

## Upstream pointers

- Repo: <https://github.com/gcla/termshark>
- Release notes: <https://github.com/gcla/termshark/releases>
- License: [MIT](https://github.com/gcla/termshark/blob/master/LICENSE)
- User guide: <https://termshark.io/>
- Author: [@gcla](https://github.com/gcla) (Graham Clark)
- Upstream dependency: [`tshark`](https://www.wireshark.org/docs/man-pages/tshark.html)
  — Wireshark CLI dissector, required at runtime.
