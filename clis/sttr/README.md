# sttr

## What it does
A **single-binary string-transformation multitool** in Go. One command,
~40 sub-transformers covering the daily grab-bag every dev keeps
re-Googling: base32/base64 encode/decode, URL encode/decode, HTML escape,
ROT13, hex ↔ bytes, MD5/SHA-1/SHA-256/SHA-512 hashes, JWT decode,
JSON ↔ YAML, JSON pretty/minify, YAML format, camelCase ↔
snake_case ↔ kebab-case ↔ PascalCase, slugify, ASCII-strip, line
sort/reverse/dedupe, word-count, Markdown-to-HTML, Lorem-Ipsum, UUID
generate, QR-code emit (as ASCII to stdout), Bcrypt hash, Caesar shift,
Morse, ZeroPad. Reads from stdin or a positional argument, writes to
stdout, composes through pipes (`cat secret.txt | sttr base64-encode |
sttr md5`). Interactive TUI mode (`sttr i`) lists every transformer with
fuzzy filter and a live preview pane — useful when you forget the
exact name.

## Why it's interesting
Different shape from `openssl` / `coreutils` / `jq` / `yq` / `xxd` /
`base64` / `uuidgen` / `qrencode` (each is one tool with one
specialty, you remember 12 different invocations and `man` pages),
from online "encode/decode" web tools (paste data into a stranger's
form — bad for secrets), from Python one-liners
(`python -c "import base64; ..."` is fine for one shot but not for a
shell pipeline). sttr is the *one binary, every text transform you
half-remembered, no network* shape: pick it specifically when you keep
a personal terminal scratchpad and want a single dependency that
covers 80% of "convert this string" asks, or in a CI step that needs
hash + encode + case-convert without pulling Python or Node. Do
**not** pick it for high-throughput streaming hashing of
multi-gigabyte files (use `sha256sum` / `b3sum` directly), for
cryptographically meaningful operations (it's a convenience layer,
not a crypto library — use `openssl` / `age` for real keys), or for
heavy structured-data work (use `jq` / `yq` / `dasel`).

## Niche category
String-transform multitool — one Go binary that absorbs base64 / hash /
case-convert / JSON-YAML / JWT-decode / QR-emit into a single piped CLI.

## Repo
https://github.com/abhimanyu003/sttr

## Version pinned
`v0.2.30` (latest tagged release, published 2025-12-25)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linux)
brew install sttr

# Go install (any platform)
go install github.com/abhimanyu003/sttr@latest

# Prebuilt binaries (Linux / macOS / Windows / FreeBSD)
# https://github.com/abhimanyu003/sttr/releases/tag/v0.2.30
curl -L https://github.com/abhimanyu003/sttr/releases/download/v0.2.30/sttr_Linux_x86_64.tar.gz \
  | tar -xz -C /usr/local/bin sttr

# Snap
sudo snap install sttr
```

## Usage examples
```sh
# Pipe-friendly: hash a secret then base64 it
echo -n "hunter2" | sttr md5 | sttr base64-encode

# Decode a JWT without copy-pasting into jwt.io
echo "eyJhbGciOi..." | sttr jwt

# Convert YAML to JSON (compact) for a curl body
sttr yaml-json < deploy.yaml | curl -X POST -d @- http://api/spec

# Rename a list of identifiers from snake_case to camelCase, in place
cat fields.txt | sttr snake-camel > fields.camel.txt

# Emit a QR code for the current Wi-Fi password as ASCII art in the terminal
echo "WIFI:S:guest;T:WPA;P:correct-horse-battery-staple;;" | sttr qr

# Interactive picker: scroll, fuzzy-match, live preview, no flags to memorize
sttr i
```
