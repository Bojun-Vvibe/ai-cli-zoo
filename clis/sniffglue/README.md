# sniffglue

> Snapshot date: 2026-05. Upstream: <https://github.com/kpcyrd/sniffglue>

**A `tcpdump`-shaped packet sniffer that runs each parser in a
seccomp-sandboxed worker thread.**
sniffglue is a single Rust binary that captures packets off a NIC
and decodes them protocol-by-protocol (Ethernet, IPv4/v6, TCP, UDP,
ARP, DNS, DHCP, TLS ClientHello, HTTP, SSDP, NTP, …) in worker
threads that have been seccomp-narrowed to almost no syscalls,
then prints a coloured one-line-per-packet summary on the main
thread. The point is not "more protocols than tcpdump" — it's "a
sniffer where a parser bug in a malformed packet can't escalate to
RCE on the host".

## Repo + version + license

- Repo: <https://github.com/kpcyrd/sniffglue>
- Latest release: **`v0.16.2`** (2026-04-19)
- License: **GPL-3.0** —
  <https://github.com/kpcyrd/sniffglue/blob/main/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `main`
- Language: Rust

## Install

```bash
# Most distros (Arch, Debian/Ubuntu, Alpine, Void) ship a package
sudo pacman -S sniffglue
sudo apt install sniffglue
sudo apk add sniffglue

# Cargo from source (needs libpcap headers + libseccomp on Linux)
cargo install sniffglue

# Sniff the default interface (needs CAP_NET_RAW or root)
sudo sniffglue

# Pick an interface and a BPF filter (same syntax as tcpdump)
sudo sniffglue -p wlan0 'port 53 or port 5353'

# Read a pcap file offline — sandbox still applies
sniffglue -r capture.pcap

# Verbose: include parsed payload fields, not just summary lines
sudo sniffglue -v -p eth0 'tcp port 80'
```

## Niche

The "**sandboxed packet sniffer for hostile networks**" slot.
`tcpdump` and `tshark` are battle-tested but they parse untrusted
input in a single privileged process; a CVE in a dissector lands as
root on the box doing the sniffing. sniffglue's design assumption
is the opposite — that a packet on the wire might be deliberately
malformed to exploit the sniffer — so:

- the main thread does the privileged `pcap_open_live` and
  immediately drops privileges;
- raw packets are handed to **worker threads under
  seccomp-bpf**, restricted to a tiny syscall whitelist (`read`,
  `write`, `brk`, `mmap`, `sigreturn`, `exit`, …) that excludes
  `open`, `execve`, `connect`, `socket`, etc.;
- if a parser panics or attempts a forbidden syscall, the worker
  is killed and the host stays intact.

Useful for:

- **Sniffing on a compromised / hostile LAN** — coffee shops,
  conference Wi-Fi, hotel networks, DEF CON, blue-team incident
  response on a known-bad segment.
- **Triaging unknown pcaps** — `sniffglue -r unknown.pcap` from a
  ticket attachment, where you'd rather not feed it to a tcpdump
  build that hasn't been patched in months.
- **Embedded / appliance sniffing** — small footprint, single
  binary, no Lua / Python plugin loader to worry about.

## Why it matters

- **seccomp-bpf sandbox per worker** — the syscall whitelist is
  defined in the source under `src/sandbox/` and is intentionally
  short; the README documents which syscalls are allowed and why,
  so you can audit the trust boundary in an afternoon.
- **Privilege drop after `pcap_open_live`** — sniffglue grabs the
  capture handle as root, then setuid/setgid drops to a `nobody`
  / `_sniffglue` user before any packet is parsed; on Linux it
  also uses `prctl(PR_SET_NO_NEW_PRIVS)`.
- **Coloured one-line summaries by default, structured fields on
  `-v`** — the output is grep-friendly (each line is one packet
  with consistent column order) and doesn't try to be a full
  protocol decoder; for that you use Wireshark on a copy of the
  pcap, in a VM, on a separate box.
- **BPF filter passthrough** — the same `tcp port 443`,
  `host 1.2.3.4`, `not port 22` syntax you already know from
  tcpdump, compiled by libpcap, applied at the kernel level so
  filtered-out packets never enter userspace.
- **Honest scope** — sniffglue does not replace Wireshark, does
  not do SSL decryption, does not maintain flow state, does not
  reassemble TCP. It is "a safer one-line-per-packet
  sniffer", and the README is explicit about that.
- **Active in 2026** — `v0.16.2` (2026-04-19) is the most recent
  release at snapshot time; the project has been maintained by
  the same author since 2017 with regular small releases.
- **Operator caveat** — sandboxing reduces blast radius; it does
  not make the parsers themselves bug-free. If your threat model
  is "nation-state malformed packet that escapes the sandbox",
  sniff inside a disposable VM regardless of which tool you use.
