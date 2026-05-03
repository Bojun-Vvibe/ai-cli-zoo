# jp

> **A standalone command-line JSON processor that
> speaks JMESPath**, the query language standardized
> by AWS for `aws --query`, OpenStack `--format`,
> Azure `--query`, and a dozen other cloud SDKs —
> read JSON on stdin, give it a JMESPath expression
> as a single argument, get the projection back as
> JSON on stdout. `jp 'foo.bar[0].baz'`,
> `jp 'Reservations[].Instances[].{ID:InstanceId,
> IP:PublicIpAddress}'`, `jp 'length(items)'` — all
> the same one-shot model as `jq`, but with the
> JMESPath grammar your AWS CLI muscle memory
> already knows. Pinned to **v0.2.1**
> ([LICENSE](https://github.com/jmespath/jp/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/jmespath/jp>

## TL;DR

You already write JMESPath every time you type
`aws ec2 describe-instances --query
'Reservations[].Instances[].{ID:InstanceId,
State:State.Name}'`. The same grammar is just as
useful when the JSON is *already on disk* — a
`kubectl get pods -o json` dump, a webhook payload
saved during a debug session, the response from a
random REST endpoint piped through `curl`. `jp` is
the dedicated CLI for exactly that case: a single
Go binary that reads JSON, evaluates one JMESPath
expression, and prints the result. No DSL to
re-learn the way `jq`'s syntax requires; if you can
say it in `--query`, you can say it in `jp`.

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install jp

# Go install (any platform with Go ≥ 1.16)
go install github.com/jmespath/jp@v0.2.1

# Pre-built binaries (release page)
#   jp-linux-amd64, jp-linux-arm64, jp-darwin-amd64,
#   jp-darwin-arm64, jp-windows-amd64, jp-freebsd-*
curl -L -o /usr/local/bin/jp \
  https://github.com/jmespath/jp/releases/download/0.2.1/jp-linux-amd64
chmod +x /usr/local/bin/jp

# verify
jp --version    # jp version 0.2.1
```

`jp` is a single static binary with zero runtime
dependencies — drop it into a container image, a
CI runner, or `~/bin` and it works.

## Use it for

```bash
# The headline use: pipe JSON in, evaluate one expression
echo '{"foo": {"bar": "baz"}}' | jp foo.bar
# "baz"

# Real-world: extract just instance IDs + state from EC2 dump
aws ec2 describe-instances --output json \
  | jp 'Reservations[].Instances[].{ID:InstanceId, State:State.Name}'

# kubectl: pod name + node mapping
kubectl get pods -o json \
  | jp 'items[].{pod:metadata.name, node:spec.nodeName}'

# Filter projection: only Running pods
kubectl get pods -o json \
  | jp 'items[?status.phase==`Running`].metadata.name'

# Flatten + sort + slice
jp -f data.json 'sort_by(items, &timestamp)[-5:].{t:timestamp, m:message}'

# Read expression from a file (long / multi-line query)
jp -e query.jmespath -f payload.json

# Compact output (default is pretty-printed)
jp -u 'foo.bar' < input.json     # unquoted strings (raw)

# Pretty-print or compact JSON
jp .                              # pretty re-emit (alias: identity)
```

The flags are deliberately few — `-f FILE` (read
JSON from file instead of stdin), `-e FILE` (read
expression from file instead of argument), `-u`
(unquoted output for raw strings), and `--ast` /
`--keys` for debugging. Everything else is
expressed *in the JMESPath itself*.

## Why include it in a CLI catalog

1. **It is the reference implementation of the
   JMESPath spec.** `jmespath/jp` lives under the
   official `jmespath` GitHub org alongside the
   spec, the test suite, and the language-level
   parsers (`go-jmespath`, `jmespath.py`,
   `jmespath.js`). When AWS, OpenStack, Azure, and
   the Kubernetes ecosystem agreed on a
   query-language standard for "filter this JSON",
   they picked JMESPath; `jp` is how you exercise
   that grammar at the shell. If you'll ever paste
   a `--query` expression, prototyping it in `jp`
   first against a saved JSON dump is faster than
   round-tripping the cloud API on every iteration.
2. **The grammar pays back across many tools, not
   just `aws`.** `jp` itself, `aws --query`,
   `az --query`, OpenStack CLIs, Splunk, Stripe CLI,
   Ansible's `json_query` filter, Kubernetes
   admission-webhook validators, and assorted
   Lambda authorizers all consume JMESPath. Time
   spent learning JMESPath via `jp` compounds; time
   spent learning a single-tool DSL does not.
3. **Single static Go binary, no runtime.** Drop
   into Alpine, distroless, scratch images, or a
   read-only initramfs without dragging in `libc`
   or a JIT. Ideal for terse `RUN` lines in
   container builds and for embedding in CI helper
   scripts that have to run on whatever the
   pipeline gave them.

For an LLM-CLI workflow, `jp` is the JMESPath half
of "extract structured fields from a model's JSON
output before piping into the next stage": the
model emits JSON, `jp 'tool_calls[].arguments'`
pulls the field your shell loop actually wants to
act on, and the result feeds the next subprocess.
Pairs naturally with [`jq`](../jq/) /
[`gron`](../gron/) / [`fx`](../fx/) when you want a
*different* query grammar for the same problem.

## Vs Already Cataloged

- **Vs [`jq`](../jq/) / [`jaq`](../jaq/) /
  [`gojq`](../gojq/):** orthogonal grammar — `jq`
  invented its own DSL (`.foo | select(.bar > 3) |
  map(.baz)`); JMESPath is a different, smaller
  spec (`foo[?bar > \`3\`].baz`) that doubles as
  the query language for `aws --query` and the
  cloud-vendor SDKs. Pick by which grammar your
  surrounding stack already uses; for AWS / Azure
  / OpenStack-heavy work, JMESPath wins on muscle
  memory.
- **Vs [`fx`](../fx/) / [`jless`](../jless/) /
  [`jnv`](../jnv/):** orthogonal interaction model
  — those are interactive JSON explorers (TUIs);
  `jp` is one-shot, batch, scriptable. Use the
  TUIs to *find* the path, then put the JMESPath
  expression in a script with `jp`.
- **Vs [`gron`](../gron/):** different paradigm —
  `gron` flattens JSON to greppable lines so you
  can use `grep` / `sed`. `jp` parses the JSON
  and evaluates a typed expression. `gron` is
  better for "I don't know the schema, let me
  grep"; `jp` is better for "I know the path, just
  give me the value".
- **Vs [`miller`](../miller/) / [`csvtk`](../csvtk/):**
  orthogonal data shape — those are tabular
  (CSV / TSV / JSONL) processors. `jp` is for
  *nested* JSON document trees. Use `mlr` /
  `csvtk` once `jp` has flattened the projection
  into a record-per-line shape.

## Caveats

- **Last release was September 2021 (v0.2.1).**
  The Go CLI itself has been stable since then —
  no functional changes since v0.2.0; v0.2.1 only
  added darwin-arm64 binaries. The JMESPath spec
  it implements has likewise been stable, and the
  underlying `go-jmespath` library still ships
  fixes. If you need active bleeding-edge
  development, look at one of the
  [JMESPath Community](https://github.com/jmespath-community)
  forks; for the canonical reference, this is it.
- **JMESPath is intentionally *less* expressive
  than `jq`.** No reduce-step, no recursion, no
  user-defined functions. The trade is
  *predictability* and cross-vendor portability.
  If your problem genuinely needs a fold,
  `jq`/`jaq` is the right tool.
- **JSON-only input / output.** No YAML, TOML, CSV,
  or msgpack. Pre-convert with `yq -o=json` /
  `dasel` / `jc` to get a JSON document `jp` can
  query.
- **Backtick literals (`` `3` ``, `` `"foo"` ``)
  inside JMESPath expressions need shell quoting
  to survive bash / zsh — usually wrap the whole
  expression in single quotes:
  `jp 'items[?count > \`3\`]'`. Same gotcha
  applies to `aws --query`, so the habit transfers.
- **Apache-2.0 license** — permissive; safe to
  redistribute the binary inside a proprietary
  container image with attribution.
