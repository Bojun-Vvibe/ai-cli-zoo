# croc

> **Single-binary, resumable, end-to-end-encrypted file transfer that
> works between any two machines (NAT, firewalls, different OSes) by
> agreeing on a short human-readable code phrase — no accounts, no
> upload to a third-party blob, no `scp` key dance.** Pinned to
> **v10.4.2**, MIT
> ([LICENSE](https://github.com/schollz/croc/blob/main/LICENSE)).

- **Repo:** https://github.com/schollz/croc
- **Latest version:** v10.4.2
- **License:** MIT (`LICENSE` at repo root, SPDX `MIT`)
- **Category:** `transfer` / `networking`
- **Language:** Go

## What it does

`croc send file.tar.gz` prints a three-word code phrase
(e.g. `1234-magnet-spirit-cargo`); on the other machine,
`croc 1234-magnet-spirit-cargo` downloads the file. Under the hood
the two peers register the code phrase against a public relay
(default `croc.schollz.com`, freely self-hostable via
`croc relay`), perform a PAKE handshake (`siec`-based password
authenticated key exchange) so a passive observer cannot decrypt the
stream even if it intercepts the relay traffic, then attempt direct
UDP / TCP hole-punching; if NAT defeats both, the relay forwards the
encrypted bytes without ever holding the key. Transfers are
chunk-resumable: kill the connection mid-100-GB upload, rerun the
same command on both ends, and `croc` picks up at the last
acknowledged chunk. Sends folders, multiple files, stdin pipes
(`tar c . | croc send`), and supports `--zip` for on-the-fly
compression and `--hash xxhash` for fast integrity checks.

## Why included

The cross-machine "I have a 4 GB tarball, the other person is on a
home network, neither of us is going to set up `scp` keys, and I'm
not uploading our build artifacts to a third-party drive" problem
has no clean answer in the standard Unix toolbox. `croc` is the
clean answer: install once (`brew install croc`, single Go binary,
no daemon), zero config, the code phrase is the only secret, the
transfer is encrypted before it touches the relay, and it resumes.
For an LLM-CLI workflow that runs an agent on a remote sandbox and
needs to ship a build artifact (or a flame graph, or a packed
context bundle) back to the operator's laptop without provisioning
SSH or S3, `croc send build.tar.zst` followed by pasting the code
phrase into chat is the lowest-ceremony path that exists.
