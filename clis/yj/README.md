# yj

> **Convert between YAML, TOML, JSON, and HCL on the command
> line** — a tiny Go binary whose name encodes its core trick:
> the first letter is the input format and the second letter is
> the output format (`yj` = YAML→JSON, `tj` = TOML→JSON,
> `jy` = JSON→YAML, `yh` = YAML→HCL, etc.). Pinned to **v5.1.0**
> (commit `500a688b5c2bd05426ac1351f93c16e32c733eb7`,
> [LICENSE](https://github.com/sclevine/yj/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/sclevine/yj>

## TL;DR

`yj` is what you reach for when you have a YAML file and a
script that only speaks JSON (or vice-versa) and you don't want
to write a Python one-liner. It is one self-contained binary, no
runtime, and the format pair lives in the command name itself —
so `cat config.yaml | yj -jc` (YAML → JSON, compact) is the
whole pipeline. It also round-trips through HCL, which makes it
useful as a glue layer between Terraform-shaped configs and
everything else.

## Install

```bash
# Homebrew (macOS / Linux)
brew install yj

# Go (any OS with toolchain)
go install github.com/sclevine/yj/v5@v5.1.0

# release binary (any OS)
curl -Lo yj "https://github.com/sclevine/yj/releases/download/v5.1.0/yj-darwin-arm64"
chmod +x yj
sudo install yj /usr/local/bin/

# verify
yj -v    # 5.1.0
```

## Sample usage

```bash
# YAML to JSON (default mode is yj — YAML in, JSON out)
yj < deployment.yaml > deployment.json

# JSON to YAML
echo '{"a":1,"b":[2,3]}' | yj -jy
# a: 1
# b:
# - 2
# - 3

# TOML to JSON, pretty-printed
yj -tj < pyproject.toml | jq .

# YAML to HCL (good for porting k8s-shaped config to a Terraform module)
yj -yh < values.yaml > values.hcl

# round-trip a config through JSON to strip comments and normalise key order
yj -yj < messy.yaml | yj -jy > clean.yaml

# combine with kubectl: extract one field from a manifest, rewrite it, push back
kubectl get cm my-config -o yaml | yj -yj | jq '.data.foo = "bar"' | yj -jy | kubectl apply -f -
```

## When to pick this

Pick `yj` when you want the lightest possible glue between
config formats in a shell pipeline — one binary, no Python, no
Node, no plugin system, format pair encoded in the command name.
Reach for `dasel` (already in this catalog) when you also need
to **query and mutate** specific paths inside the document
without piping through `jq`; reach for `yq` (already in this
catalog) when you want a `jq`-syntax superset that natively
understands YAML anchors, multi-doc streams, and in-place edits.
`yj` stops at "convert the whole document" and stays small.

## License

Apache-2.0 — see [LICENSE](https://github.com/sclevine/yj/blob/main/LICENSE).
