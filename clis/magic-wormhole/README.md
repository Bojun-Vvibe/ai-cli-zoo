# magic-wormhole

- **Repo:** https://github.com/magic-wormhole/magic-wormhole
- **Version:** 0.19.2 (latest PyPI release, 2025-08)
- **License:** MIT ([LICENSE](https://github.com/magic-wormhole/magic-wormhole/blob/master/LICENSE))
- **Language:** Python
- **Install:** `brew install magic-wormhole` · `pipx install magic-wormhole`
  · `pip install --user magic-wormhole` · also packaged in Debian, Ubuntu,
  Fedora, Arch, NixOS

## What it does

`wormhole` is a one-line, end-to-end-encrypted file / directory /
text transfer between two terminals that may be on entirely
different networks, behind entirely different NATs, with no shared
account, no preconfigured keypair, no cloud bucket, and no
SaaS login. Sender runs `wormhole send file.tar.gz` and the binary
prints a short human-pronounceable code phrase like
`7-crossover-clockwork`. Receiver on any other machine — laptop,
phone via Termux, server over SSH — runs `wormhole receive
7-crossover-clockwork` and the file lands in their cwd, integrity
checked. Under the hood the code phrase is a
[PAKE](https://en.wikipedia.org/wiki/Password-authenticated_key_agreement)
(specifically SPAKE2) input: the two ends derive a strong shared
key from a small low-entropy phrase without ever sending the
phrase, the key, or any password-equivalent over the wire. A small
public *rendezvous server* (`wormhole.leastauthority.com` by
default) brokers the introduction; if both ends can reach each
other directly, transfer goes peer-to-peer; otherwise it relays
through a separate *transit relay* — both servers see only
ciphertext, neither sees the file content or the code. The wire
protocol is documented and re-implemented in Rust
([magic-wormhole.rs](https://github.com/magic-wormhole/magic-wormhole.rs))
and Go, so the receiver does not have to use the same binary as
the sender.

## When to pick it / when not to

Pick `wormhole` whenever the question is "I have a thing on machine
A, I want it on machine B, both belong to humans, and I do not want
to think about S3 / scp keys / Dropbox / Google Drive / Slack 25 MB
limits / the corporate proxy that strips attachments". One code
phrase, no account, end-to-end encrypted, works across NATs without
a VPN, kills the connection after the transfer completes. Concrete
fits: shipping a 200 MB log bundle from a customer's laptop to your
debugging session; moving an SSH private key from your old machine
to your new one without bouncing it through email; sending a
just-built binary from a Linux dev box to a colleague's Mac during
a pair-debug; quick `wormhole send --text "$LONG_TOKEN"` from a
phone to a server's terminal so you do not paste a secret into
chat. Pair with [`age`](../age/) when the recipient must be a
specific *identity* (encrypt to their pubkey first, then ship the
ciphertext over wormhole — defense in depth against a future
rendezvous-server compromise) and with [`croc`](../croc/) as the
single-binary peer if a Python / pipx install is not available on
either end (croc has the same model with a different code-phrase
shape and an embedded relay).

Skip `wormhole` for fully unattended machine-to-machine transfers
on a schedule — the code-phrase model assumes a human is reading
one end aloud or pasting it, not a cron job; for that, use `scp` /
`rsync over ssh` / `rclone` between accounts you actually own.
Skip it as your *backup* tool — there is no incremental sync, no
deduplication, no retention; pair with [`restic`](../restic/) or
[`kopia`](../kopia/) for that. Skip it for very large transfers
on flaky links — there is resume support
(`wormhole send --resume` / `--receive --resume`) since 0.13, but
a 200 GB dataset over a residential link is not where this tool
shines; rsync over ssh on a proper bastion is. And note the
default rendezvous and transit relays are run by Least Authority
on a best-effort basis; if you depend on this in a product, run
your own (`wormhole-mailbox-server`, `wormhole-transit-relay`)
and point both ends at it via `--relay-url` / `--transit-helper`.

## Example invocations

```bash
# Send a single file
wormhole send build/release.tar.gz
# → prints: Wormhole code is: 7-crossover-clockwork

# Receive on the other end
wormhole receive 7-crossover-clockwork

# Send a whole directory (auto-zipped end-to-end-encrypted in transit)
wormhole send ./project/

# Send a one-off text snippet (no file written on sender)
wormhole send --text "rotate-this-token-and-then-burn-it"
# Receiver sees the text on stdout, nothing touches disk except history.

# Resume a transfer that died half-way (since 0.13)
wormhole send --resume big.iso
wormhole receive --resume <code>

# Self-hosted relays (paranoid mode; both ends must agree)
wormhole \
  --relay-url=ws://wormhole.example.com:4000/v1 \
  --transit-helper=tcp:transit.example.com:4001 \
  send ./payload.tar

# Pipe-from-stdin trick: encrypt with age, ship over wormhole, decrypt on the other end
tar c ./secrets/ \
  | age -r age1exampleidentity... \
  | wormhole send --text - >/dev/null   # paste code into other terminal
# Receiver:
wormhole receive <code> | age -d -i ~/.config/age/keys.txt | tar x
```
