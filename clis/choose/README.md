# choose

> **A human-friendly and fast alternative to `cut` (and
> sometimes `awk`)** — a small Rust CLI that picks fields from
> stdin using intuitive Python-style indices and ranges, with
> sensible whitespace defaults so you stop fighting `cut -d' '
> -f3`. Pinned to **v1.3.7**
> ([LICENSE](https://github.com/theryangeary/choose/blob/master/LICENSE),
> GPL-3.0).

Source: <https://github.com/theryangeary/choose>

## TL;DR

`choose` is `cut` with the rough edges sanded off. Whitespace
is the default delimiter (any run of whitespace, like `awk`),
indices start at 0, negative indices count from the end (`-1`
is last), and ranges use Python-style `0:2` (or open-ended
`2:` and `:3`). `ps aux | choose 1 10:` prints PID and full
command. `df -h | choose -1` prints the mountpoint column. No
`-d' '` boilerplate, no "wait, did `-f1` mean field 1 or
field 0?" — the indices match what your brain already does in
Python / Rust slice syntax.

## Install

```bash
# Homebrew (macOS / Linux)
brew install choose-rust

# Cargo (any OS with a Rust toolchain)
cargo install choose

# Arch (AUR)
yay -S choose

# verify
choose --version    # choose 1.3.7

# pick the third whitespace-separated field
echo "a b c d e" | choose 2          # c

# slice: fields 1 through 3 inclusive
echo "a b c d e" | choose 1:3        # b c d

# negative index: last field
df -h | choose -1
```

## License

GPL-3.0 — see
[LICENSE](https://github.com/theryangeary/choose/blob/master/LICENSE).
Copyleft; fine to use as a CLI in any pipeline, but don't
vendor the source into a proprietary binary.

## Niche It Fills

**`cut` rewritten with sane defaults for humans.** POSIX `cut`
makes you spell `-d' ' -f3` and only accepts a single
delimiter character (no runs of whitespace), and field numbers
start at 1. `choose` defaults to "any whitespace" like `awk`,
indexes from 0 like every modern language, and accepts
Python-style ranges so you don't have to mentally translate
between slice syntax and shell field syntax.

## Why it pairs with coding agents

Agents constantly emit shell pipelines that need to grab a
single column from `kubectl get pods`, `docker ps`, `git log
--pretty`, `ls -l`, etc. `choose` makes the resulting one-liner
shorter, clearer, and less error-prone than `awk '{print $3}'`
or `cut -d' ' -f3`, which means agents are less likely to
generate broken pipelines around irregular whitespace. Easier
to read in agent transcripts too — `choose 2` is unambiguous;
`awk '{print $3}'` requires a mental parse.

## Vs Already Cataloged

- **Vs `sd` (already cataloged):** `sd` is a `sed`
  replacement (substitution); `choose` is a `cut` /
  field-selection replacement. Sister tools, complementary
  scopes.
- **Vs `awk`:** `awk` is a full programming language; `choose`
  is intentionally just field selection. If you need `BEGIN
  { ... }` or arithmetic, use `awk`. For "give me column 3"
  pipelines, `choose` is cleaner.

## Caveats

- **GPL-3.0 means viral linking.** Fine for shell pipelines
  (you're invoking a separate process), but don't link the
  Rust crate into a proprietary binary.
- **Not a CSV parser.** No quoted-field handling; if your
  input has `"a, b", c, d`, use a real CSV tool (`xsv` /
  `qsv`).
- **Active but slow-moving.** Project is stable, not racing —
  v1.3.7 covers the field-selection scope completely, expect
  patch releases not new features.
