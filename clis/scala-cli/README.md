# scala-cli

> **Single-binary launcher for Scala — REPL, scripts, tests,
> single-file programs, and packaged apps without a `build.sbt`.**
> Reads `//> using` directives at the top of `.scala` files to
> declare deps, JVM flags, Scala version, and platform (JVM /
> Scala.js / Scala Native), so a one-file Scala script is a
> first-class artifact you can `run`, `test`, `package`, and
> `publish` with one CLI. Now the official Scala runner shipping
> as the `scala` command in Scala 3.5+. Pinned to **v1.13.0**
> ([LICENSE](https://github.com/VirtusLab/scala-cli/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/VirtusLab/scala-cli>

## TL;DR

`scala-cli run hello.scala` boots a Scala 3 program in seconds
with no `build.sbt`, no `project/` dir, and no Maven coordinates
to memorise — `//> using dep "com.lihaoyi::os-lib:0.10.7"` at
the top of the file is the dependency declaration. The same
binary handles `repl`, `test`, `compile`, `package` (fat JAR,
native image via GraalVM, Docker, deb / rpm / pkg, JS bundle),
`publish` (Maven Central via Sonatype, gpg-signed), `fmt`
(scalafmt), `fix` (scalafix), and `setup-ide` (writes a
`.bsp/scala-cli.json` so Metals / IntelliJ pick it up). It is
the Scala-side answer to `cargo` / `go run` / `deno run` for
single-file programs, scripts, and small services.

## Install

```bash
# Homebrew (macOS / Linux)
brew install Virtuslab/scala-cli/scala-cli

# Coursier
cs install scala-cli

# Direct installer (Linux / macOS)
curl -sSLf https://scala-cli.virtuslab.org/get | sh
```

## Example

```scala
// hello.scala
//> using scala "3.4.2"
//> using dep "com.lihaoyi::os-lib:0.10.7"
//> using dep "com.lihaoyi::upickle:3.3.1"

@main def hello(name: String) =
  val cwd = os.pwd
  println(ujson.write(ujson.Obj("hello" -> name, "cwd" -> cwd.toString)))
```

```bash
# Run the single file
scala-cli run hello.scala -- world

# Repl with the file's deps preloaded
scala-cli repl hello.scala

# Package as a GraalVM native binary
scala-cli --power package --native-image hello.scala -o hello

# Publish to Maven Central from the same file
scala-cli --power publish hello.scala
```

## When to use

- You want to write a Scala script, micro-service, or
  proof-of-concept in one file without scaffolding an sbt /
  Mill project.
- You teach Scala and want students to type `scala-cli run
  Foo.scala` instead of explaining the sbt build model on day 1.
- You need to package a small Scala tool as a fat JAR, native
  image, or distro package and don't want to wire up sbt-assembly
  / sbt-native-packager by hand.
- You maintain a polyglot monorepo where a few Scala utilities
  live alongside Go / Rust / Python — `scala-cli` keeps each
  utility self-contained instead of requiring a project skeleton.

## When NOT to use

- You have an existing multi-module sbt or Mill build with
  custom plugins, source generators, and cross-compilation
  matrices — stay on the existing build tool; `scala-cli` is
  optimised for the small-and-self-contained end of the spectrum.
- You need the full sbt plugin ecosystem (sbt-protobuf,
  sbt-buildinfo, custom task graphs) — `scala-cli` exposes a
  smaller surface by design.
- You are looking for a generic JVM build tool — use Mill,
  Gradle, or Maven; `scala-cli` is Scala-first.
