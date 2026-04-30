# oras

- **Repository:** https://github.com/oras-project/oras
- **Latest version:** v1.3.2
- **License:** Apache-2.0 — verified at [`LICENSE`](https://github.com/oras-project/oras/blob/main/LICENSE) (raw: https://raw.githubusercontent.com/oras-project/oras/main/LICENSE)
- **Niche:** OCI registry / container-image tooling

## What it does

`oras` ("OCI Registry As Storage") is a CLI for pushing and pulling
**arbitrary artifacts** — not just container images — to and from any
OCI-compliant registry. It speaks the OCI distribution and image specs
directly, so you can store Helm charts, WASM modules, SBOMs, signatures,
ML models, config bundles, or plain tarballs alongside your images
without standing up a separate artifact store.

Typical flows:

```
oras push registry.example.com/my-artifact:1.0 ./model.bin:application/vnd.example.model
oras pull registry.example.com/my-artifact:1.0
oras attach --artifact-type sbom/cyclonedx myimage:tag sbom.json
oras discover myimage:tag
```

## Why interesting

Once you stop thinking of an OCI registry as "the place container images
live" and start thinking of it as a generic content-addressable artifact
store, a lot of supply-chain problems simplify: SBOMs, attestations,
signatures, and provenance can all live next to the artifact they
describe, addressed by the same digest, served by the same auth, mirrored
by the same registry replication. `oras` is the lingua franca for that
pattern and is the upstream the rest of the OCI artifact ecosystem
(notation, cosign attach, in-toto attestations) standardized around.

## When to reach for it in an AI/coding workflow

- Distributing fine-tuned model weights or datasets through your existing
  registry instead of a separate object store.
- Shipping prompt bundles, eval fixtures, or agent skill packs as
  versioned, digest-addressable artifacts.
- Attaching SBOMs / signatures / build provenance to images produced by
  agent-driven build pipelines.
