# gomplate

> **Go-template-powered CLI renderer that turns any text file into
> a templated artifact, pulling data from env vars, JSON/YAML/TOML
> files, HTTP, AWS/GCP/Vault secret stores, and arbitrary commands —
> a "jinja for shell" without a Python runtime.** Pinned to **v5.0.0**,
> MIT
> ([LICENSE](https://github.com/hairyhenderson/gomplate/blob/main/LICENSE)).

- **Repo:** https://github.com/hairyhenderson/gomplate
- **Latest version:** v5.0.0
- **License:** MIT (`LICENSE` at repo root, SPDX `MIT`)
- **Category:** `dev-experience` / `config-templating`
- **Language:** Go
- **Install:** `brew install gomplate` or
  `go install github.com/hairyhenderson/gomplate/v4/cmd/gomplate@latest`

## What it does

`gomplate` is a single-binary Go-template renderer with a much
larger function library than stock `text/template`: string ops
(`strings.ReplaceAll`, `regexp.Replace`), collection helpers
(`coll.Merge`, `coll.JSONPath`), crypto (`crypto.SHA256`,
`crypto.Bcrypt`), filesystem (`file.Read`, `file.Walk`), network
(`net.LookupIP`, `net.LookupTXT`), time, math, base64, UUID, and
~200 more. Inputs are declared as **datasources** with a URL-style
scheme: `--datasource cfg=file:///etc/cfg.yaml`,
`--datasource api=https://example.com/api/x?type=application/json`,
`--datasource sec=vault:///secret/data/app`,
`--datasource s=aws+ssm:///prod/db`, `--datasource c=cmd:///bin/uptime`.
Inside the template, `{{ (ds "cfg").database.host }}` walks the
parsed structure. Outputs can be one file (`--in/--out`), a tree
(`--input-dir/--output-dir`), or many-to-one with `--output-map`.
`gomplate -c .=./config.yaml -f tmpl.j2 -o rendered.conf` is the
typical one-liner.

## Why included

Most "render a config from a template" flows reach for `envsubst`
(env vars only, no logic), `jinja2-cli` (drags in Python +
`pyyaml`), or `helm template` (locks you into Kubernetes chart
shape). `gomplate` is the middle ground: one static binary, no
runtime, but with conditionals, loops, and pluggable datasources
including HTTP / Vault / cloud secret managers — so the same tool
renders an `nginx.conf` on a bare VM, an `.env` file in a CI job,
and a templated PR body in a GitHub Action. For an LLM-CLI
workflow that asks an agent to "generate the deployment manifest
from this JSON spec", `gomplate` is the deterministic rendering
layer the agent can call instead of string-concatenating YAML in
its own head.
