# grpcurl

> **`curl` for gRPC services — a single Go binary that introspects,
> calls, and pretty-prints any gRPC endpoint over HTTP/2 without you
> writing a `.proto` file or generating a stub.** Pinned to **v1.9.3**
> ([LICENSE](https://github.com/fullstorydev/grpcurl/blob/master/LICENSE),
> MIT).

Source: <https://github.com/fullstorydev/grpcurl>

## TL;DR

`grpcurl` is what you reach for when somebody hands you a hostname,
a port, and the words "it speaks gRPC" and you need to ship a fix
in the next ten minutes. It uses gRPC's **server reflection**
protocol (or a local `.proto` / protoset file you point it at) to
discover services, methods, and message shapes at runtime, then
lets you invoke any unary or streaming RPC with a JSON request body
and prints the JSON response. No code generation, no language
runtime, no SDK. One static binary, talks HTTP/2, supports TLS,
mTLS, OAuth bearer tokens, custom metadata headers, gzip
compression, plaintext for local dev, and connection over Unix
domain sockets. Authored at FullStory and now the de-facto debug
tool for any service mesh that exposes gRPC.

## Install

```bash
# Homebrew (macOS / Linux)
brew install grpcurl

# Go (any platform with Go >=1.22)
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@v1.9.3

# Pre-built binary (releases page)
curl -sSL https://github.com/fullstorydev/grpcurl/releases/download/v1.9.3/grpcurl_1.9.3_linux_x86_64.tar.gz \
  | tar -xz -C /usr/local/bin grpcurl

# Verify
grpcurl -version    # grpcurl 1.9.3
```

## License

MIT — see
[LICENSE](https://github.com/fullstorydev/grpcurl/blob/master/LICENSE).
Permissive, vendoring inside another product is fine, no copyleft
on the wire data you exchange. The Go module path is
`github.com/fullstorydev/grpcurl` and the binary command lives
under `cmd/grpcurl`.

## One Concrete Example

```bash
# 1. list the services exposed by a server with reflection enabled
grpcurl localhost:50051 list
# greet.Greeter
# grpc.health.v1.Health
# grpc.reflection.v1alpha.ServerReflection

# 2. list methods on a specific service
grpcurl localhost:50051 list greet.Greeter
# greet.Greeter.SayHello
# greet.Greeter.SayHelloStream

# 3. describe a message to see the field shape
grpcurl localhost:50051 describe greet.HelloRequest
# greet.HelloRequest is a message:
# message HelloRequest {
#   string name = 1;
#   int32 retries = 2;
# }

# 4. invoke a unary RPC with a JSON body
grpcurl -plaintext -d '{"name":"world","retries":3}' \
  localhost:50051 greet.Greeter/SayHello
# {
#   "message": "hello world (3)"
# }

# 5. invoke against a TLS endpoint with bearer auth + custom metadata
grpcurl \
  -H "authorization: Bearer $TOKEN" \
  -H "x-request-id: $(uuidgen)" \
  -d '{"id":"acct_123"}' \
  api.example.com:443 billing.v1.AccountService/GetAccount

# 6. server-streaming RPC: read one JSON message per server frame
grpcurl -plaintext -d '{"channel":"prices"}' \
  localhost:50051 market.Feed/Subscribe

# 7. client-streaming or bidi: pipe newline-delimited JSON on stdin
echo -e '{"name":"a"}\n{"name":"b"}\n{"name":"c"}' | \
  grpcurl -plaintext -d @ localhost:50051 greet.Greeter/SayHelloStream

# 8. server doesn't have reflection? point at a compiled protoset
protoc --descriptor_set_out=greet.protoset --include_imports greet.proto
grpcurl -protoset greet.protoset -plaintext \
  -d '{"name":"world"}' localhost:50051 greet.Greeter/SayHello

# 9. health check (every gRPC server should expose this)
grpcurl -plaintext localhost:50051 grpc.health.v1.Health/Check
```

## Niche It Fills

**Be `curl` for gRPC services so you can poke a backend, write a
shell script against it, or smoke-test a deploy without generating
a single line of stub code.** REST debugging has `curl`, `httpie`,
`xh`, `hurl`. gRPC, until `grpcurl`, required either generating a
client in some language, writing a tiny program, and running it —
or wiring up a heavyweight GUI. `grpcurl` collapses that to a
one-liner that takes JSON in and JSON out, which means you can
pipe it through `jq`, paste it into a runbook, embed it in a CI
smoke test, or use it from a Bash for-loop just like `curl`.

## Why use it

Three things `grpcurl` does that the alternatives do not:

1. **Server reflection means zero schema management.** If the
   server has the reflection service registered (one line in most
   gRPC setups), `grpcurl` discovers every service and message at
   call time. You don't need to know the `.proto` file path,
   don't need to keep it in sync with the deployed binary, don't
   need to regenerate clients on every schema bump. The schema
   travels with the server, and `grpcurl` reads it on demand.
2. **JSON in, JSON out — pipe-friendly by construction.** Every
   request body is JSON (with `-d`), every response frame is
   pretty-printed JSON. That means `grpcurl ... | jq '.items[]'`
   works the same as it does for REST, and you can build runbooks
   and smoke tests out of plain shell pipelines instead of
   compiled clients.
3. **One static binary, no language runtime.** `grpcurl` is a Go
   binary, ~20 MB, zero dependencies. You can scp it onto a
   bastion host, drop it into an Alpine container, ship it to a
   Kubernetes node via a debug pod, or vendor it into a CI image
   without dragging in protoc, a JVM, Node, or a Python venv.

For an on-call engineer who needs to verify "is the billing service
actually answering?" at 3 AM, the workflow is `grpcurl -plaintext
$HOST:$PORT grpc.health.v1.Health/Check` and read the JSON. No
codegen, no SDK, no editor open.

## Vs Already Cataloged

- **Vs [`hurl`](../hurl/):** different protocol families. `hurl`
  scripts HTTP/1.1 + HTTP/2 + HTTP/3 request/response flows in a
  declarative `.hurl` file with assertions, captures, and chained
  requests — its sweet spot is REST and GraphQL integration tests.
  `grpcurl` is single-call ad-hoc gRPC; there is no scripted
  multi-request format and no built-in assertion DSL. Pair them:
  `hurl` for REST suites, `grpcurl` for the gRPC half of the same
  service.
- **Vs [`httpstat`](../httpstat/):** orthogonal layer. `httpstat`
  is a `curl` wrapper that visualizes TCP / TLS / HTTP-1
  connection timings; it doesn't speak gRPC framing. `grpcurl`
  speaks gRPC over HTTP/2 but doesn't break down per-phase
  timings. To debug "is this slow at the wire or the app?" run
  `grpcurl -v` for protocol details and `httpstat` for the
  underlying TCP/TLS handshake.
- **Vs [`curlie`](../curlie/) / `curl`:** `curl` can technically
  send HTTP/2 frames to a gRPC port, but you have to manually
  encode the length-prefixed protobuf payload — useless in
  practice. `grpcurl` is the gRPC-shaped tool that fills the same
  ergonomic slot `curlie` fills for REST.

## Caveats

- **Reflection must be enabled on the server, or you must supply
  a `.proto` / protoset.** Production servers often disable
  reflection by default for surface-area reasons. Workarounds:
  (a) enable reflection in non-prod environments, (b) compile the
  schema with `protoc --descriptor_set_out=schema.protoset
  --include_imports` and pass `-protoset`, or (c) point at a
  directory of `.proto` files with `-import-path` + `-proto`. Do
  not assume reflection is on in prod.
- **`-plaintext` disables TLS — use only for localhost/dev.** The
  default is TLS; flipping `-plaintext` is the most common
  copy-paste mistake. For self-signed certs in staging, prefer
  `-insecure` (skip verify) over `-plaintext` so the wire is
  still encrypted.
- **JSON encoding loses some protobuf precision.** `int64` and
  `uint64` are emitted as JSON strings (per the canonical
  protobuf-JSON mapping) to avoid JavaScript number precision
  loss; `bytes` fields are base64; well-known types like
  `google.protobuf.Timestamp` are RFC 3339 strings. If you are
  diffing wire-level bytes, prefer `-emit-defaults` and read the
  protobuf-JSON spec carefully.
- **Bidi streaming over stdin is line-delimited JSON, not
  protobuf framing.** That is convenient for shell use but means
  you cannot replay a captured wire dump directly — you have to
  decode it to JSON first.
- **Not a load tester.** `grpcurl` makes one call per invocation.
  For RPS / latency benchmarking against gRPC, reach for
  `ghz` (separate tool), or [`k6`](../k6/) with its gRPC module
  for a scripted load profile.
