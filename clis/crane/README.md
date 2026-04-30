# crane

A no-nonsense CLI for interacting with remote OCI/Docker container image
registries without ever touching a local Docker daemon. Part of the
`go-containerregistry` toolkit.

## Repo

- URL: https://github.com/google/go-containerregistry
- Subcommand source: `cmd/crane/`

## Version

- v0.20.6 (release tag, late 2025)

## License

- Apache-2.0
- License file path in upstream: `LICENSE`

## Install

```sh
go install github.com/google/go-containerregistry/cmd/crane@latest
# or download a static binary from the GitHub Releases page
```

## Use case category

Container image plumbing / supply chain tooling. Sits between a registry
client (skopeo) and a registry server, focused on fast scripted
workflows in CI.

## Example usage

```sh
# Copy an image between registries with no daemon, preserving digest
crane copy gcr.io/distroless/static:nonroot ghcr.io/example/static:nonroot

# List the tags on a remote repo
crane ls ghcr.io/example/static
```

Expected behavior: `crane copy` streams blobs directly between the two
registries (no local pull/push), prints the destination digest on
success, and exits non-zero on auth or manifest errors. `crane ls`
prints one tag per line on stdout.

## Strengths

- Daemonless: never spawns or talks to dockerd / containerd; works
  cleanly inside hermetic CI runners and distroless build images.
- Surgical: separate verbs for `manifest`, `config`, `blob`, `digest`,
  `mutate`, `append`, `flatten` — you can rewrite an image's config
  without re-pulling layers.
- Extremely small footprint (single static Go binary) and stable
  exit codes, which makes it safe to script around in `Makefile`s
  and shell pipelines.

## Limitations

- Not a builder. It can mutate and append layers, but it will not
  evaluate a Dockerfile — pair it with `ko`, `apko`, `buildah`, or
  `BuildKit` for that.
- Error output for registry auth issues can be terse; you often need
  `--verbose` plus the registry's own logs to diagnose 401/403s.

## Comparison

Versus `cosign` (already in this zoo): `cosign` focuses on signing,
verifying, and attesting OCI artifacts, while `crane` focuses on
moving and mutating the artifacts themselves — the two are
complementary halves of a registry-side supply chain workflow.

## Notes on positioning in the zoo

The zoo already covers image-supply-chain reading (`syft`), scanning
(`trivy`), and signing (`cosign`); `crane` fills the missing
"plumbing" slot — the verb most CI scripts actually want when they
just need to copy an image, retag it, or read its manifest without
spinning up a daemon.

## When to reach for it

- You need to retag or mirror an image across registries from a CI
  job that has no Docker socket.
- You want to inspect a remote image's config or layers (e.g. to
  assert it is distroless, or to extract a label) without pulling
  the full image to disk.
- You're building higher-level supply-chain tooling and want a
  stable, daemonless OCI client library to embed (the same Go
  module powers `ko`, `crane`, and parts of `cosign`).
