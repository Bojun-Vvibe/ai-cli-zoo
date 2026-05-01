# grpcui

> **Interactive web UI for any gRPC server, the way `curl` + a
> browser is for HTTP** — a single Go binary that points at a
> gRPC endpoint, reads the service descriptors via reflection (or
> a `.proto` / protoset file), and serves a local web form where
> every RPC, every message field, and every metadata header is a
> typed input you can fill in and fire. Pinned to **v1.4.3**
> (released 2025-04-04,
> [LICENSE](https://github.com/fullstorydev/grpcui/blob/master/LICENSE),
> MIT).
>
> Source: <https://github.com/fullstorydev/grpcui>

## TL;DR

`grpcui` is the answer when you have a gRPC service and you do
not want to hand-write a `grpcurl` invocation every time you poke
it. Same author and same descriptor pipeline as
[`grpcurl`](https://github.com/fullstorydev/grpcurl), but instead
of a CLI it spins up a browser-based form: pick a service, pick a
method, fill the request message field-by-field with the right
types pre-validated, set request metadata, hit **Invoke**, see
response message + trailers + status. Streaming RPCs (server,
client, bidi) are first-class — you can send multiple request
messages and watch the response stream arrive in the page. Drops
in for QA, sales demos, on-call debugging, and the "I just want
to click the API" moments where curl ergonomics break down.

## Install

```bash
# Homebrew (macOS / Linux)
brew install grpcui

# Go (any OS, requires Go toolchain)
go install github.com/fullstorydev/grpcui/cmd/grpcui@latest

# Static binary (any OS)
# https://github.com/fullstorydev/grpcui/releases

# verify
grpcui -version    # grpcui 1.4.3
```

## Examples

```bash
# point at a server that has gRPC reflection enabled, open the UI
grpcui -plaintext localhost:50051
# -> "gRPC Web UI available at http://127.0.0.1:NNNNN/"

# TLS endpoint with reflection
grpcui grpc.example.com:443

# server has no reflection — feed it the protoset directly
protoc --descriptor_set_out=api.protoset --include_imports api.proto
grpcui -plaintext -protoset api.protoset localhost:50051

# or feed raw .proto files plus their import path
grpcui -plaintext \
  -import-path ./proto \
  -proto api/v1/orders.proto \
  localhost:50051

# bind to a fixed port so a teammate can hit the same UI
grpcui -plaintext -port 8910 -bind 0.0.0.0 localhost:50051

# preset request metadata (auth, tenant, trace headers)
grpcui -plaintext \
  -rpc-header 'authorization: Bearer eyJhbGciOi...' \
  -rpc-header 'x-tenant: acme' \
  localhost:50051

# embed the UI as a handler in your own Go HTTP server
# (library mode: import "github.com/fullstorydev/grpcui/standalone")
```

## Use when

- You are demoing a gRPC service to someone who does not want to
  install a CLI and learn protobuf JSON encoding — a URL + a form
  is a much shorter path to "show me the API works".
- You are doing **on-call gRPC debugging** and you need to issue
  one-off RPCs with varying field values without rebuilding a
  `grpcurl` JSON blob each time; the form remembers your last
  inputs per method across reloads.
- You need to exercise **streaming RPCs by hand** (send N client
  messages, watch the server stream back) — `grpcurl` can stream
  but the ergonomics are awkward; `grpcui` makes it click-driven.
- Your service ships **gRPC reflection** in non-prod and you want
  a zero-config explorer your QA team can run locally against
  staging without any `.proto` files on their machine.
- Pair with [`grpcurl`](../grpcurl/) (same descriptor stack,
  scriptable for CI / load-test seeding),
  [`buf`](../buf/) (lint, breaking-change detection, and
  descriptor generation that feeds `grpcui -protoset`),
  [`bombardier`](../bombardier/) /
  [`vegeta`](../vegeta/) (after grpcui proves the request shape,
  drive load against the same RPC),
  [`kubectl-tree`](../kubectl-tree/) +
  [`stern`](../stern/) (find the gRPC pod, tail logs while you
  click in grpcui).

Skip `grpcui` when the workflow is fully scripted (one-shot CI
checks, golden-file replay, load tests) — `grpcurl` is the right
tool there. Skip when the server has no reflection *and* no
protoset *and* the proto sources are not nearby; `grpcui` needs a
descriptor source to render the form. Not for HTTP/REST; pair
with [`httpie`](../httpie/) / [`xh`](../xh/) /
[`posting`](../posting/) for those.
