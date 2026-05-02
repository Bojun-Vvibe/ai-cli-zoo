# witness

> **A pluggable CNCF-sandbox CLI that wraps any build / test /
> deploy command and emits a signed in-toto attestation
> describing what happened — `witness run -s build -k cosign.key
> -- go build ./...` records the command, the git commit, the
> file hashes of every input and output, the environment, the
> OCI images touched, and signs the bundle so a downstream
> verifier (`witness verify`) can reject artifacts whose
> provenance doesn't match a declared policy.** Pinned to
> **v0.11.0** (released 2026-04-14),
> [LICENSE](https://github.com/in-toto/witness/blob/main/LICENSE),
> Apache-2.0.

Source: <https://github.com/in-toto/witness>

## TL;DR

`witness` is the practical, command-wrapping CLI that turns
the in-toto attestation framework from a spec into a daily-
driver supply-chain control. You don't change the build —
you prefix it: `witness run -s build -- make release` runs
`make release` exactly as before, but witness sidecars the
process with a stack of **attestors** (small probes that
record one fact each: `git`, `gitlab`, `github`, `commandrun`,
`environment`, `material` (input file hashes), `product`
(output file hashes), `oci`, `slsa`, `sbom`, `vex`, `maven`,
`jwt`, `aws-iid`, `gcp-iit`) and signs the resulting
in-toto Statement (a JSON document of subject + predicate)
with cosign / KMS / PKCS#11 / file keys / Sigstore Fulcio.
The signed attestation (`*.witness.json`) ships alongside
the artifact; later, `witness verify -p policy.yaml -k
public.key -f artifact.tar.gz` walks a **policy** that
declares which attestors must have run, in which order, by
which key, with which constraints (e.g. "build must follow
test", "no environment leaked AWS_*"), and exits non-zero if
the artifact's attestation chain doesn't satisfy it. The
result: a verifier in CI / admission controller / release
pipeline can mechanically reject builds that bypassed the
test step or were signed by an unauthorised key — SLSA-style
provenance without writing your own framework.

## Install

```bash
# Homebrew (macOS / Linux)
brew install witness

# Static binary from GitHub Releases (linux / macOS / windows, amd64 / arm64)
curl -fsSL -o /tmp/witness.tar.gz \
  https://github.com/in-toto/witness/releases/download/v0.11.0/witness_0.11.0_darwin_arm64.tar.gz
tar -xzf /tmp/witness.tar.gz -C /tmp
sudo install /tmp/witness /usr/local/bin/

# Go install
go install github.com/in-toto/witness@v0.11.0

# Verify
witness version    # 0.11.0

# Generate a signing key (or use cosign / KMS / Fulcio instead)
openssl genpkey -algorithm ed25519 -out signer.key
openssl pkey -in signer.key -pubout -out signer.pub
```

## One Concrete Example

```bash
# Wrap a multi-step build, sign each step's attestation,
# then declare a policy that requires both steps were run
# in order by the same key — verify before publishing.

# 1. Wrap the test step
witness run \
  --step test \
  --signer-file-key-path signer.key \
  --outfile test.witness.json \
  --attestations git,environment,commandrun,material,product \
  -- go test ./...

# 2. Wrap the build step
witness run \
  --step build \
  --signer-file-key-path signer.key \
  --outfile build.witness.json \
  --attestations git,environment,commandrun,material,product,oci \
  -- go build -o ./bin/server ./cmd/server

# 3. Declare a policy: the artifact must have a signed
#    `test` attestation followed by a signed `build` from
#    the same key, both on the same git commit
cat > policy.yaml <<'EOF'
expires: "2027-01-01T00:00:00Z"
steps:
  test:
    attestations:
      - type: https://witness.dev/attestations/commandrun/v0.1
      - type: https://witness.dev/attestations/git/v0.1
    functionaries:
      - type: publickey
        publickeyid: KEY_ID_HERE
  build:
    attestations:
      - type: https://witness.dev/attestations/commandrun/v0.1
      - type: https://witness.dev/attestations/material/v0.1
      - type: https://witness.dev/attestations/product/v0.1
    functionaries:
      - type: publickey
        publickeyid: KEY_ID_HERE
publickeys:
  KEY_ID_HERE:
    keyid: KEY_ID_HERE
    key: |
      -----BEGIN PUBLIC KEY-----
      ...contents of signer.pub...
      -----END PUBLIC KEY-----
EOF

# Compute the policy key id witness expects
KEY_ID=$(witness keyid --publickey signer.pub)
sed -i '' "s/KEY_ID_HERE/$KEY_ID/g" policy.yaml

# 4. Sign the policy itself (verifiers trust the signed policy)
witness sign \
  --signer-file-key-path signer.key \
  --infile policy.yaml \
  --outfile policy.signed.json

# 5. Verify the artifact against the policy + its attestations
witness verify \
  --policy policy.signed.json \
  --publickey signer.pub \
  --artifactfile ./bin/server \
  --attestations test.witness.json \
  --attestations build.witness.json
# exit 0 = policy satisfied, safe to publish
# exit 1 = some required attestor missing / wrong key / wrong commit
```

In CI the same shape becomes a gate: the publish job runs
`witness verify` and the registry push only happens on exit 0.

## License

[Apache-2.0](https://github.com/in-toto/witness/blob/main/LICENSE),
SPDX `Apache-2.0`. CNCF sandbox project under in-toto.

## Niche / positioning

The **command-wrapper for in-toto attestations** — pick
`witness` over rolling your own `sha256sum` + `gpg --sign`
script when you want a typed, signed, policy-verifiable
record of *how* an artifact was built, not just a hash of the
output. Pick over [`cosign`](../cosign/) when the question is
"what process produced this?" instead of "is this exact blob
signed by this key?" — the two compose: `witness` writes the
attestation, `cosign attest` (or witness's own signers) ships
it next to the OCI image, and a verifier validates both.
Pick over [`slsa-verifier`](../slsa-verifier/) when you need
to author your own provenance + policy (slsa-verifier only
verifies attestations produced by the SLSA GitHub generator);
witness is the producer side and supports SLSA + custom
predicates. Pick over [`syft`](../syft/) when you want
provenance (how the build ran) instead of an SBOM (what's
inside the artifact) — they're complementary: a witness
attestation often carries an SBOM as one of its attestors.
Skip when your only need is "sign this binary" (use cosign
directly) or when the org has standardised on Tekton Chains
or GitHub's built-in attestation generator and a separate
wrapper would just duplicate the trust chain.
