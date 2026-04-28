# jaq

- **Repo:** https://github.com/01mf02/jaq
- **Version:** v3.0.0 (latest stable, 2026-03)
- **License:** MIT ([LICENSE](https://github.com/01mf02/jaq/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install jaq` · `cargo install --locked jaq` · `nix run nixpkgs#jaq` · binary releases on the GitHub release page

## What it does

`jaq` is a from-scratch reimplementation of `jq` in Rust, focused on three
things the original is weak on: speed (typically 2x–30x faster on large
inputs because it compiles the filter to a small bytecode and runs a
non-allocating evaluator), correctness (it ships a written semantics and a
test suite that pins behaviour for tricky cases — `null` propagation,
`limit`, recursive `..`, slice indexing on strings vs arrays — that have
historically diverged across `jq` 1.5 / 1.6 / 1.7), and a saner default for
streaming input. The CLI surface is a deliberate near-superset of `jq`'s:
the same `.foo.bar`, `.[]`, `select()`, `map()`, `group_by`, `to_entries`,
`reduce`, `def`, variable bindings, modules, and most stdlib functions
(`length`, `keys`, `paths`, `walk`, `split`, `sub`/`gsub`, `tojson`/`fromjson`,
`now`, etc.) work unchanged. Differences are mostly additive: `jaq` adds
`abs`, `nan`, `infinite`, regex via the `regex` crate (which is faster and
more predictable than oniguruma), and the `--in-place` flag for editing a
JSON file without a `>` shell-redirection footgun. The `--slurp` (`-s`),
`--raw-input` (`-R`), `--raw-output` (`-r`), `--compact-output` (`-c`),
`--null-input` (`-n`), `--arg`, `--argjson`, `--slurpfile`, and `--from-file`
flags all behave identically. NDJSON is the streaming default — feed a 50 GB
log file through `jaq '.user.id'` and it processes one record at a time
without buffering the whole input, which is the practical perf story (the
microbenchmarks are nice; the ability to `cat huge.ndjson | jaq -c …` on a
laptop is the actual win).

## When to pick it / when not to

Pick `jaq` when you already write `jq` and either (a) the input file is
large enough that wall-clock matters, (b) you have hit one of the documented
`jq` quirks (the `limit` infinite-loop bug, `walk` mutation order surprises,
`@base32d` returning bytes rather than UTF-8) and want documented behaviour,
or (c) you are shipping a tool that needs to embed a `jq`-shaped query
language and would rather not depend on the C `jq` build (jaq's library
crate, `jaq-core`, is a clean Rust dependency). The `--in-place` flag plus
the speed make `jaq` a reasonable substitute for ad-hoc Python `json.load /
modify / json.dump` scripts. Pair with [`gron`](../gron/) when you do not
yet know the JSON shape (gron flattens to grep-friendly paths, jaq queries
once you know what you want), with [`ripgrep`](../ripgrep/) over NDJSON
where the structure is shallow enough that line-grep beats parsing, and
with [`fx`](https://github.com/antonmedv/fx) (not in this catalog) when you
want an interactive JSON explorer instead of a batch query language.

Skip it when the input is YAML / TOML / XML — `jaq` is JSON-only; reach for
[`yq`](https://github.com/mikefarah/yq) or [`dasel`](https://github.com/tomwright/dasel)
for those formats. Skip it when you depend on a niche `jq` extension that
`jaq` has not yet implemented (`@uri` edge cases, `getpath` semantics on
missing keys, the `$__loc__` debug variable) — the compatibility table in
the `jaq` README lists current gaps; if your filter sits in production CI
and you cannot afford a behaviour change, pin `jq`. Skip it for streaming
JSON where individual records are 100 MB+ — both `jq` and `jaq` parse a
record at a time and a 100 MB record blows past sensible memory budgets;
that is a `simdjson` / Rust-`serde-json-streaming` shaped problem. For
pretty-printing-only with no transformation, `python -m json.tool` and
`jq .` are fine — you do not need a new binary for that.

## Example invocations

```bash
# Drop-in replacement for jq on a local file
jaq '.users[] | .email' users.json

# NDJSON streaming: one request log per line, extract status codes
cat access.ndjson | jaq -r '.status' | sort | uniq -c

# Build a request body from CLI args
jaq -n --arg name "Bojun" --argjson age 33 '{name: $name, age: $age}'

# Edit a JSON file in place (no shell-redirect footgun)
jaq --in-place '.version = "2.1.0"' package.json

# Compose with chained filters using a pipeline
jaq '.items | map(select(.active)) | group_by(.tier) | map({tier: .[0].tier, n: length})' inventory.json

# Run a saved filter from a file (multi-line, with comments and `def`)
jaq --from-file ./queries/top-errors.jq logs.ndjson

# Slurp a stream of objects into one array, then sort by timestamp desc
jaq -s 'sort_by(.ts) | reverse | .[:10]' events.ndjson
```

Sample saved filter (`queries/top-errors.jq`) for reference:

```jq
# Top 10 error endpoints in an NDJSON access log, by p95 latency
def p95: sort | .[(length * 0.95) | floor];

select(.status >= 500)
| {endpoint: .path, latency_ms}
| group_by(.endpoint)
| map({
    endpoint: .[0].endpoint,
    n: length,
    p95_ms: (map(.latency_ms) | p95)
  })
| sort_by(-.p95_ms)
| .[:10]
```
