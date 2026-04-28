# gron

- **Repo:** https://github.com/tomnomnom/gron
- **Version:** v0.7.1 (latest stable, 2022-04 — stable, no churn since)
- **License:** MIT ([LICENSE](https://github.com/tomnomnom/gron/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install gron` · `go install github.com/tomnomnom/gron@latest` · binary releases on the GitHub release page

## What it does

`gron` ("grep on JSON") flattens any JSON document into one assignment per
line, so every leaf value gets a self-describing path. Pipe in
`{"users":[{"name":"ada","tags":["admin","early"]}]}` and you get back:

```
json = {};
json.users = [];
json.users[0] = {};
json.users[0].name = "ada";
json.users[0].tags = [];
json.users[0].tags[0] = "admin";
json.users[0].tags[1] = "early";
```

Now `grep`, `rg`, `awk`, `sed`, `cut`, and even your editor's plain text
search work on JSON without learning a query DSL. The companion subcommand
`gron --ungron` (alias `gron -u`) round-trips: feed it any subset of those
assignment lines (in any order, with gaps) and it reconstructs valid JSON,
so a typical workflow is `curl … | gron | grep something | gron -u` to get a
filtered JSON document back. It streams, has no schema, no config, and no
runtime — single static binary, ~2 MB.

## When to pick it / when not to

Pick `gron` when you need to **discover the shape** of an unfamiliar JSON
blob (an undocumented API response, a CloudWatch event, a webpack stats
dump), when you need to **diff two JSON documents in a way that survives key
reordering** (`diff <(curl A | gron | sort) <(curl B | gron | sort)`), or
when you want to grep a deeply-nested field without writing a `jq` selector
("which paths in this 5 MB blob mention `email`?" → `gron file.json | grep
-i email`). It is also the fastest way to teach JSON paths to someone who
already knows shell — every line *is* a path. Pairs naturally with
[`jq`](../jq/) (structural transforms),
[`jless`](../jless/) (interactive viewer), and
[`dasel`](../dasel/) (multi-format selector).

Skip it when you already know the exact path and just need a value — `jq
-r '.users[0].name'` is shorter. Skip it for very large files where you only
need one field — `jq` streams better. Skip it when the data is YAML / TOML
/ XML — use `dasel` or `yq`. And remember `--ungron` is a *reconstruction*,
not a patch: it replaces the document, not merges into it.

## Example invocations

```bash
# Discover the shape of an API response
curl -s https://api.github.com/repos/tomnomnom/gron | gron | head -20

# Find every path mentioning a substring (case-insensitive)
curl -s https://api.github.com/repos/tomnomnom/gron | gron | grep -i license

# Order-independent diff of two JSON documents
diff <(curl -s https://api.github.com/repos/tomnomnom/gron | gron | sort) \
     <(curl -s https://api.github.com/repos/tomnomnom/gron | gron | sort)

# Filter and reconstruct: keep only owner.* fields
curl -s https://api.github.com/repos/tomnomnom/gron \
  | gron \
  | grep '^json.owner' \
  | gron --ungron
```
