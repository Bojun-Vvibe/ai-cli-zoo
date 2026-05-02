# evans

- **Repo:** https://github.com/ktr0731/evans
- **Version:** v0.10.11 (latest stable, February 2023; project still
  maintained, slow release cadence)
- **License:** MIT ([LICENSE](https://github.com/ktr0731/evans/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew tap ktr0731/evans && brew install evans` · `go install github.com/ktr0731/evans@latest` (Go 1.20+) · `docker run --rm -v "$(pwd):/mount:ro" ghcr.io/ktr0731/evans:latest` · static binaries on the GitHub release page · binary name is `evans`

## What it does

`evans` is a **gRPC client with two front-ends**: an interactive REPL
mode where it walks you through a service field-by-field with tab
completion, and a stateless CLI mode where each invocation sends one
request and writes the JSON response to stdout, in the spirit of
`curl` for HTTP. It speaks vanilla gRPC (protobuf over HTTP/2),
gRPC-Web (against `improbable-eng/grpc-web` proxies), TLS with custom
roots and client certs, GZIP compression, and all four streaming
shapes (unary / client-streaming / server-streaming / bidi). It can
discover services either by reading `.proto` files directly
(`--proto api.proto --path ./proto`) or by talking to a server that
has gRPC reflection enabled (`-r`) — in the reflection case you don't
need any local proto files at all.

The REPL is the part that makes `evans` distinctive. After
`evans -r repl` you get commands like `show package`, `package api`,
`show service`, `desc SimpleRequest`, `service Example`,
`call Unary`, all with completion, and `call` then prompts you for
each field interactively (`name (TYPE_STRING) =>`). Headers can be
set per-session (`header foo=bar`) and persisted into `.evans.toml`
in the project root, so a teammate cloning the repo gets the same
default proto path / package / service / headers without re-typing.

## When to pick it / when not to

Pick `evans` when you are **building or debugging a gRPC service
interactively**: you don't yet know all the field names, you want
completion to teach you the schema, and you want headers / metadata
to persist between calls so you can iterate. The REPL is also how
you most cheaply explore an unfamiliar gRPC service that has
reflection turned on — `evans -r --host=svc.example.test --port=443
--tls repl` and you can `show service` your way around the API
without ever opening the `.proto` files.

Pick the CLI mode (`evans -r cli call api.Example.Unary < req.json`)
when you want a `curl`-shaped invocation in a runbook or a CI step:
input on stdin or `-f file.json`, output JSON on stdout, exit code
nonzero on RPC failure, pipe to `jq` for assertions.

Pick something else when:

- You only need **one-shot calls in a script** and don't care about
  schema exploration — [`grpcurl`](https://github.com/fullstorydev/grpcurl)
  is leaner, ships fewer features, and is the de-facto standard for
  CI scripts. `evans` CLI mode covers that case but `grpcurl` is the
  more conservative choice.
- You need **load testing** — `evans` is a client, not a load
  generator. Use [`ghz`](https://github.com/bojand/ghz) for that.
- You need **gRPC-Web with TLS** — `evans` supports gRPC-Web but
  not over TLS yet (per the upstream README), so for a
  browser-shaped TLS gRPC-Web flow you may need a different tool.
- You need a **GUI** with proto file watching, request history,
  collections — that's [`Postman`](https://www.postman.com/) or
  [`Kreya`](https://kreya.app/), not a terminal REPL.

## Why it matters in an AI-native workflow

A growing share of "internal API" surface is gRPC, and LLM agents
that get pointed at a gRPC service have historically had a bad time:
no `curl` equivalent, schema lives in `.proto` files the agent has
to parse first, and the wire format is binary so the agent can't
just inspect a packet capture. `evans` collapses all of that into
two affordances the agent can actually use:

1. **Reflection-based discovery without local files.**
   `evans -r --host=... cli list` enumerates services,
   `evans -r cli list api.Example` enumerates methods,
   `evans -r cli desc api.Example.Unary` prints the schema in a
   form the model can read directly into context. No filesystem
   munging, no protoc invocation.
2. **JSON in, JSON out.** The CLI mode takes JSON on stdin and
   writes JSON on stdout, so the agent can construct a request as a
   string, pipe it in, parse the response with `jq`, and feed the
   result back into its own reasoning loop without ever touching a
   protobuf serializer.

That second property — JSON-in, JSON-out, exit-code-on-failure — is
exactly the contract LLM tool-calling expects, which is why `evans`
tends to drop cleanly into agent toolchains where `grpcurl` would
otherwise need a wrapper.

## Example invocations

```bash
# REPL against a server with reflection enabled.
evans -r --host=api.example.test --port=443 --tls repl

# REPL against a server using a local .proto file.
evans --proto api/api.proto --path ./api repl

# One-shot CLI call: stdin JSON, stdout JSON.
echo '{"name":"ada"}' | evans -r --host=api.example.test --port=50051 \
    cli call api.Example.Unary

# CLI call from a file, with a custom metadata header.
evans -r --host=api.example.test --port=50051 \
    --header='authorization=Bearer $TOKEN' \
    cli call -f request.json api.Example.Unary

# Enumerate services, then methods on a service.
evans -r --host=api.example.test --port=50051 cli list
evans -r --host=api.example.test --port=50051 cli list api.Example

# Describe a method's request / response schema.
evans -r --host=api.example.test --port=50051 cli desc api.Example.Unary

# Repeat the previous unary call from inside the REPL.
> call --repeat Unary
```

## Caveats

- Releases are infrequent (latest tag is from early 2023) but the
  repo is still active on `master` and the maintainer accepts PRs;
  treat it as "stable, slow-moving" rather than "abandoned".
- gRPC-Web support exists but is **not** TLS-capable as of 0.10.x —
  if you need TLS gRPC-Web, terminate TLS at a proxy and point
  `evans` at the cleartext side.
- `go install github.com/ktr0731/evans@latest` is explicitly
  discouraged by upstream because it bypasses the built-in
  self-update channel; prefer Homebrew or the GitHub release
  binaries if you want `evans update` to work later.
- The REPL prompt's CTRL-C behavior is non-obvious: it skips the
  rest of the fields in the current message rather than aborting the
  call. Use CTRL-D to abort. This is a frequent first-time-user
  papercut.
