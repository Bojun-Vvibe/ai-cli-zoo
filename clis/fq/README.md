# fq

- **Repo:** https://github.com/wader/fq
- **Latest version:** v0.17.0 (release `v0.17.0`)
- **HEAD on `master`:** `153a846`
- **License:** MIT — [LICENSE](https://github.com/wader/fq/blob/master/LICENSE)
- **Category:** binary inspection / data-format query

A **`jq` for binary formats** — same query language, same filter
pipeline, but the input tree is a parsed binary file. Out of the box
`fq` ships ~120 decoders covering containers (MP4 / Matroska / FLV /
OGG), audio / video codecs (AAC / FLAC / MP3 / Opus / Vorbis /
H.264 / H.265 / AV1), images (JPEG / PNG / GIF / TIFF / BMP /
WebP), archives (ZIP / TAR / GZIP / BZIP2), executables (Mach-O / ELF
/ PE), TLS / X.509, ASN.1 BER / DER, BSON / MessagePack / CBOR /
Protobuf / Avro, BitCoin / Ethereum on-chain blobs, and primitives
like `bits` / `bytes` / `hex`. Every decoded value carries its
**bit-range** in the source file, so a query like
`.tracks[].sample_table.sample_size` doubles as a **byte-accurate
hex viewer** — `fq -d mp4 'dv .tracks[0].edit_list' file.mp4`
prints the parsed edit list *and* the raw bytes underlying it,
side-by-side. The REPL (`fq -i`) gives tab-completion across the
decoded tree, so you can spelunk an unknown MP4 / PNG / TLS handshake
without opening a hex editor.

## Install

```bash
# macOS
brew install fq

# Arch (AUR)
yay -S fq

# Linux / macOS / Windows — Go install
go install github.com/wader/fq@latest

# Linux — prebuilt binary
curl -L -o fq.tar.gz https://github.com/wader/fq/releases/download/v0.17.0/fq_0.17.0_linux_amd64.tar.gz
tar xzf fq.tar.gz && sudo mv fq /usr/local/bin/

# verify
fq -v
```

## One usage example

```bash
# Show every keyframe (sync sample) timestamp in an MP4, formatted as JSON:
fq '
  .tracks[]
  | select(.media.handler.handler_type == "vide")
  | .sample_table.sync_sample_box.entries[]
  | { sample: ., kind: "keyframe" }
' video.mp4
```

For a quick "what is this binary?" check, just run `fq . unknown.bin`
— `fq` auto-probes the format, prints a one-line summary, and you can
drill in from there with `.` queries inside `fq -i unknown.bin`.

## When to pick `fq` (vs alternatives)

- Pick **`fq`** when the input is a **binary** format with structure
  (an MP4 you need to inspect, a PNG with a suspicious chunk, a TLS
  capture you want to walk field-by-field, a Protobuf / MessagePack
  payload from a debugger) and you already speak `jq`. The "decode
  once, query the tree, jump to underlying bytes" loop is the
  killer-app — no other tool offers the same `jq`-language fidelity
  *over* a parsed binary tree with bit-precise back-references.
- Pick [`jq`](../jq/) / [`jaq`](../jaq/) when the input is plain
  JSON; `fq` *can* handle JSON, but `jq` is the canonical tool with
  the largest community and best documentation for that case.
- Pick [`hexyl`](../hexyl/) when you only need a colourised hex dump
  with no parsing — `hexyl` is the right tool when you do not yet
  know the format, or the format is bespoke and has no decoder.
- Pick `xxd` / `od` for the lowest-common-denominator hex dump on a
  server that has neither `fq` nor `hexyl`.
- Pick [`gron`](../gron/) when the input is JSON and you want
  *grepable line-oriented* output for shell pipelines; `fq` is the
  opposite philosophy (decoded tree + `jq` queries).
- Pick `wireshark` / [`termshark`](../termshark/) when the binary is
  a *packet capture* and you want protocol dissectors with reassembly
  and follow-stream — `fq` decodes individual TLS records, but pcap
  reassembly is wireshark's lane.
