# httpyac

**Repo:** https://github.com/AnWeber/httpyac
**Version:** 6.16.7
**License:** MIT — [LICENSE](https://github.com/AnWeber/httpyac/blob/main/LICENSE)
**Language:** TypeScript (Node.js)

## What it does

`httpyac` is a CLI for executing `*.http` and `*.rest` files — the same
plain-text request format popularized by JetBrains and the REST Client
VS Code extension — with first-class support for HTTP, gRPC, WebSocket,
and MQTT in a single binary. Variables, environments, response
assertions, and JS hooks are all expressible directly inside the
`.http` file, which means the request collection, the test, and the
fixture all live next to the code in a diff-able text format. No GUI
required, no proprietary collection format, no cloud sync.

## Install

```bash
# npm
npm install -g httpyac

# Homebrew
brew tap AnWeber/httpyac
brew install httpyac

# or single-file binary from
# https://github.com/AnWeber/httpyac/releases
```

## Real usage example

Create `api.http`:

```http
@host = https://api.github.com

###
# @name getRepo
GET {{host}}/repos/AnWeber/httpyac
Accept: application/vnd.github+json

?? status == 200
?? body license.spdx_id == "MIT"

###
# @ref getRepo
GET {{host}}/repos/AnWeber/httpyac/releases/latest

?? status == 200
{{
  exports.tag = response.parsedBody.tag_name;
  console.log("latest tag:", exports.tag);
}}
```

Run it:

```bash
httpyac send api.http --all
# executes both requests in order, prints the assertion results,
# and emits the captured `tag` variable for downstream chaining
```

Other useful invocations:

```bash
# pick one named request out of a file
httpyac send api.http --name getRepo

# environment switching (reads .env / dotenv-style files)
httpyac send api.http --env prod --env secrets

# JSON-formatted CI output
httpyac send api.http --output json --bail
```

## Why it's interesting (orthogonal niche)

The CLI catalog already has plenty of one-shot HTTP clients
(`xh`, `curlie`, `httpie`, `hurl`, `posting`) but `httpyac` occupies a
specific orthogonal niche: it's the only tool here that takes the
**JetBrains `.http` file format** as a first-class input *and* extends
it to gRPC / WebSocket / MQTT in the same syntax. That makes it the
natural CLI runner for teams whose request library already lives in
`.http` files inside the repo (because IDE engineers are editing them
all day) — CI just runs `httpyac send **/*.http` against a staging
environment, the same file the developer was just hand-firing in the
IDE. Inline JS hooks plus `??` assertions also turn each `.http` file
into a runnable contract test, which is a niche `hurl` covers for
HTTP-only and `xh` does not cover at all.
