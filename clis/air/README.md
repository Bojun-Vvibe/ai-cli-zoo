# air

> **Live reload for Go applications during development** —
> watches your source tree, rebuilds the binary on every save,
> kills the previous process, restarts the new one, and streams
> stdout/stderr to your terminal with color-coded prefixes,
> all driven by a single `.air.toml` config. Pinned to
> **v1.65.1**
> ([LICENSE](https://github.com/air-verse/air/blob/master/LICENSE),
> MIT).

Source: <https://github.com/air-verse/air>

## TL;DR

`air` is the Go equivalent of `nodemon` or Django's `runserver`
auto-reloader: while you edit `.go` files, `air` notices,
runs `go build -o ./tmp/main`, kills the running `./tmp/main`,
starts the new one, and pipes its output back to your shell.
Configuration is a `.air.toml` at the repo root: which
extensions to watch (`.go`, `.tpl`, `.html`), which directories
to ignore (`tmp`, `vendor`, `node_modules`), what build command
to run (defaults to `go build -o ./tmp/main .`, but anything
shell works — `go build -tags=dev`, `templ generate && go build`,
`go run github.com/swaggo/swag/cmd/swag init && go build`),
how to invoke the binary (`./tmp/main` plus args), and how
long to debounce (`delay = 1000` ms) so a save-burst from
gofmt or a multi-file refactor doesn't trigger N rebuilds.
Build errors are printed inline, the previous binary keeps
running, and the next save retries.

## Install

```bash
# Homebrew
brew install air

# Go install
go install github.com/air-verse/air@v1.65.1

# install script
curl -sSfL https://raw.githubusercontent.com/air-verse/air/master/install.sh | sh -s -- -b $(go env GOPATH)/bin

# Static binary from GitHub Releases
curl -fsSL -o /usr/local/bin/air \
  https://github.com/air-verse/air/releases/download/v1.65.1/air_1.65.1_$(uname -s | tr A-Z a-z)_arm64
chmod +x /usr/local/bin/air

# verify
air -v    # 1.65.1
```

## One Concrete Example

```bash
# scaffold a config in your project root
cd ./api
air init    # writes .air.toml with sensible defaults

# tweak it
cat > .air.toml <<'TOML'
root = "."
tmp_dir = "tmp"

[build]
  cmd = "templ generate && go build -o ./tmp/server ./cmd/server"
  bin = "./tmp/server"
  full_bin = "PORT=8080 LOG_LEVEL=debug ./tmp/server"
  include_ext = ["go", "templ", "html", "sql"]
  exclude_dir = ["tmp", "vendor", "node_modules", "testdata"]
  exclude_regex = ["_test\\.go$", "_templ\\.go$"]
  delay = 800
  stop_on_error = false
  send_interrupt = true
  kill_delay = "2s"

[log]
  time = true

[color]
  build = "yellow"
  runner = "green"
  app = "magenta"

[misc]
  clean_on_exit = true
TOML

# run
air
# building...
# running...
# [GIN] 2026/04/29 - 10:14:22 | 200 | listening on :8080
# (edit a handler, save)
# building...
# running...
```

The trick that makes `air` work well with code generators
(templ, sqlc, swag) is the `cmd` field: it can be any shell
pipeline, so the generator runs *before* `go build` on every
reload, and you don't have to remember to re-run `templ
generate` by hand after editing a `.templ` file.

## Niche It Fills

**The default "Go dev loop" tool.** Stdlib Go has no built-in
watch mode (`go run` runs once and exits). Without something
like `air`, every save is a manual `Ctrl-C ; go run ./cmd/srv`.
With it, the loop is "save → wait ~1s → hit refresh." For
HTTP servers, gRPC servers, CLI daemons, and code-generated
codebases (templ, sqlc, swag), `air` is the canonical answer.

## Vs Already Cataloged

- **Vs [`watchexec`](../watchexec/) / [`entr`](../entr/):**
  general-purpose watchers — they re-run any command on file
  change. You can absolutely build the same loop with
  `watchexec -e go -r -- go run ./cmd/server`. `air` adds
  Go-aware defaults (the `tmp/` build cache convention,
  `_test.go` exclusion, signal handling that gives the running
  HTTP server time to drain), a single `.air.toml` per repo
  that the team checks in, and color-coded log streams. Pick
  `watchexec` for polyglot repos with one rule for everything;
  pick `air` for Go-shaped repos.
- **Vs [`reflex`](https://github.com/cespare/reflex):** older
  Go file-watcher, similar concept. `reflex` is a regex →
  command mapper (`reflex -r '\.go$' -s -- go run ./cmd/srv`),
  no config file. `air` ships a config file plus richer
  process-management defaults (graceful shutdown, post-build
  hook, separate log streams).
- **Vs [`gow`](https://github.com/mitranim/gow):** another
  Go watcher; CLI-flag-driven, no config file. Lighter
  install, fewer features, no graceful shutdown story.
- **Vs [`tilt`](../tilt/) / [`skaffold`](../skaffold/):** those
  are *cluster*-level dev loops (build container → push →
  deploy → restart pod). `air` is the *process*-level dev
  loop on your laptop. They're not competitors — `air` runs
  inside the container that `tilt` rebuilds, or `air` runs
  on your host while `tilt` watches a different service.
- **Vs `go run -gcflags='all=-N -l'` + dlv:** `air` runs the
  built binary directly, so attaching a debugger means
  configuring `air` to invoke `dlv exec ./tmp/main --headless
  --listen=:2345 --` instead of running the binary directly.
  Possible, but more friction than running `dlv debug` in a
  separate terminal. Use `air` for the edit-save-refresh loop
  and reach for `dlv` separately when you need breakpoints.

## Caveats

- Build errors leave the *previous* binary running by default
  (`stop_on_error = false`) — usually what you want, but it
  means you might be testing against stale code if you didn't
  notice the build failed. Watch the terminal, or set
  `stop_on_error = true` to crash hard on bad builds.
- The watched extension list is conservative — `air init`
  watches only `.go` by default. If you generate Go from
  `.proto`, `.sql`, `.templ`, `.html`, etc., add them to
  `include_ext` *and* make the build `cmd` re-run the
  generator, or you'll see stale output without knowing why.
- File-system watching uses fsnotify under the hood — on
  macOS, large vendored trees can hit `kqueue` open-file
  limits. Either keep `vendor/` in `exclude_dir` (the default)
  or `ulimit -n 4096` your shell.
- Signal handling assumes the child responds to SIGINT
  promptly — long-running cleanup (database connection
  drain, in-flight request finish) past `kill_delay` will be
  SIGKILLed. For HTTP servers, ensure `http.Server.Shutdown`
  is wired to SIGINT and tune `kill_delay` if needed.
- It's a *development* tool — no one runs `air` in production.
  The build artifact at `./tmp/main` is unstripped, debug-friendly,
  and rebuilt every save; production is `go build -ldflags='-s
  -w' -o /usr/local/bin/server` baked into a container image.
