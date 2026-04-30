# gojq

> **Pure-Go reimplementation of `jq`** — a drop-in `jq`-syntax
> processor written from scratch in Go, with a stable embeddable
> library API (`github.com/itchyny/gojq`) used by tools like
> `fq`, `usql`, and many TUIs that need a query DSL without
> shelling out to C `jq`. Pinned to **v0.12.19** (commit
> `b7ebffbfc038677520df0bae4c8c2d877f88ffea`,
> [LICENSE](https://github.com/itchyny/gojq/blob/main/LICENSE),
> MIT).

Source: <https://github.com/itchyny/gojq>

## TL;DR

`gojq` is what you reach for when you want `jq`-the-language but
not `jq`-the-C-binary — for example on a stripped-down container
image where you can drop a single static Go binary but cannot
add an `apt` dependency, or inside another Go program where you
want to evaluate user-supplied filters without forking a child
process. It tracks upstream `jq` syntax closely (manual is the
single source of truth), adds a few quality-of-life touches
(better error messages with location pointers, YAML I/O via
`--yaml-input` / `--yaml-output`), and ships as a tiny
self-contained CLI plus a clean library.

## Install

```bash
# Homebrew (macOS / Linux)
brew install gojq

# Go (any OS with toolchain)
go install github.com/itchyny/gojq/cmd/gojq@v0.12.19

# release binary (any OS)
curl -LO https://github.com/itchyny/gojq/releases/download/v0.12.19/gojq_v0.12.19_darwin_arm64.tar.gz
tar xf gojq_v0.12.19_darwin_arm64.tar.gz
sudo install gojq_v0.12.19_darwin_arm64/gojq /usr/local/bin/

# verify
gojq --version    # gojq 0.12.19
```

## Sample usage

```bash
# same surface as jq
echo '{"a":[1,2,3]}' | gojq '.a | map(.*10)'
# [10,20,30]

# YAML in, JSON out (no need to pipe through yj first)
gojq --yaml-input '.spec.template.spec.containers[].image' < deployment.yaml

# YAML in, YAML out — keep the round-trip native
gojq --yaml-input --yaml-output '.metadata.labels.env = "prod"' < deployment.yaml

# null-input mode — generate JSON from scratch
gojq -n '{ts: now, host: env.HOSTNAME, msg: "hello"}'

# read filter from a file (good for long pipelines)
gojq -f extract.jq < events.ndjson
```

```go
// embed the engine in a Go program — no exec, no parsing fork
import "github.com/itchyny/gojq"

q, _ := gojq.Parse(".items[] | .name")
iter := q.Run(payload) // payload is a Go any/map[string]any tree
for {
    v, ok := iter.Next()
    if !ok { break }
    fmt.Println(v)
}
```

## When to pick this

Pick `gojq` when you want `jq` semantics but a **single static
binary** (great for `FROM scratch` containers, alpine images,
locked-down CI) or when you are writing a Go program that needs
to evaluate user-supplied `jq` filters without `os/exec`. Stick
with the original C `jq` (in this catalog elsewhere) when you
care about the very last percent of compatibility with obscure
filters, when you need the lowest possible memory footprint on
a 50 GB NDJSON stream, or when your distro already ships it and
you do not want a second binary on `$PATH`.

## License

MIT — see [LICENSE](https://github.com/itchyny/gojq/blob/main/LICENSE).
