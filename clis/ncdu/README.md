# ncdu

NCurses Disk Usage — fast, interactive disk usage analyzer in the terminal.

- **Repo**: https://code.blicky.net/yorhel/ncdu (mirror: https://dev.yorhel.nl/ncdu)
- **Version**: 2.5
- **License**: MIT — `COPYING`
- **Language**: Zig (2.x); C (1.x legacy)
- **Category**: file-mgmt / disk-usage / observability

## Why it's interesting

Scans a directory tree once and gives you a navigable TUI sorted by size,
with per-item delete, recompute, and export-to-JSON actions. Drop-in
replacement for `du | sort` workflows when hunting down which `node_modules`,
HuggingFace cache, or model checkpoint dir blew up your laptop SSD.
The 2.x rewrite in Zig is dramatically faster than the original C version
on large trees and uses constant-bounded memory.

## Install

```sh
brew install ncdu
# or on Debian/Ubuntu
sudo apt install ncdu
```

## Example invocation

```sh
ncdu ~/.cache/huggingface
```
