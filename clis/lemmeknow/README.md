# lemmeknow

> **"What is this string?" — feed it any
> mystery token (a hash, an API key, a UUID,
> a base64 blob, a JWT, a credit-card-shaped
> number, an IP, a phone number, a crypto
> address) and it returns a ranked list of
> what the string most likely *is*.** A
> Rust port of PyWhat with ~800+ regex
> identifiers. Pinned to **v0.8.0**
> ([LICENSE](https://github.com/swanandx/lemmeknow/blob/main/LICENSE),
> MIT).

Source: <https://github.com/swanandx/lemmeknow>

## TL;DR

Half of incident-response and CTF work is
"someone pasted me a string, what is it?".
You stare at `f7c3bc1d808e04732adf679965ccc34c
a7ae3441` and try to remember whether SHA-1
is 40 hex chars or 32. `lemmeknow` short-
circuits that: pipe the string in, get back
"SHA-1 hash", "Git commit SHA", "Gravatar
avatar URL fragment" — ranked by likelihood,
with a `Description` and `URL` link for each
guess. Built on a curated regex database
shared with the PyWhat project (~800
identifiers covering crypto wallets, JWTs,
AWS keys, social-security numbers, MAC
addresses, RFC2822 dates, MIME types, and a
long tail of ID formats).

## Install

```bash
# cargo
cargo install lemmeknow

# Pre-built binary (Linux / macOS / Windows)
curl -L -o lemmeknow https://github.com/swanandx/lemmeknow/releases/download/v0.8.0/lemmeknow-macos
chmod +x lemmeknow && sudo mv lemmeknow /usr/local/bin/

lemmeknow --version    # lemmeknow 0.8.0
```

## Use it for

```bash
# Identify a single string
lemmeknow "f7c3bc1d808e04732adf679965ccc34ca7ae3441"
# → SHA-1, Git commit hash, ...

# Read from a file (or stdin)
lemmeknow -f leaked-tokens.txt
cat suspicious.log | lemmeknow

# JSON output for pipelines
lemmeknow -o json "ya29.a0AfH6SMBx..." | jq '.[].Identifier.Name'

# Filter to a specific identifier family (e.g. only crypto)
lemmeknow --tags Cryptocurrency "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa"

# Verbose mode prints description + reference URL
lemmeknow -v "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

## Why include it in a CLI catalog

1. **It collapses a recurring 30-second
   task to 1 second.** "Is this a UUIDv4 or
   a ULID?" "Is this an AWS access key or a
   GitHub PAT?" "Is this a SHA-256 or
   SHA3-256?" Stop squinting; just ask.
2. **The identifier database is the value.**
   ~800 hand-curated regexes covering API
   keys (AWS, Stripe, GitHub, Slack, Twilio,
   …), crypto wallets (BTC, ETH, XMR, …),
   document IDs (passports, SSN, NHS,
   Aadhaar, …), and dev artifacts (Git SHA,
   npm tarball URL, JWT, Base64-encoded
   PNGs). Hand-rolling these is what makes
   the tool actually useful.
3. **Fast and offline.** Single static Rust
   binary, no network calls — safe to run on
   secrets you can't send to a web service
   like cyberchef.org or dcode.fr.
4. **Scriptable.** JSON output makes it a
   building block for log triage, secret-
   scanner pre-classification, or CTF
   automation.

## Vs Already Cataloged

- **Vs [`noseyparker`](../noseyparker/) /
  [`gitleaks`](../gitleaks/):** different
  job. Those scan *codebases / git
  histories* for secret-shaped strings and
  flag findings. `lemmeknow` classifies *one
  string at a time* across a much broader
  catalog (not just secrets — also IDs,
  hashes, encoded blobs). Use a scanner to
  find candidates, then `lemmeknow` to
  understand what each candidate is.
- **Vs `file` / [`hexyl`](../hexyl/):** those
  identify *binary blobs / file types*;
  `lemmeknow` identifies *short text
  tokens*. Complementary.

## Caveats

- Pure regex matching → false positives are
  inevitable. A 40-char hex string matches
  SHA-1 and Git SHA and "any 40-char hex"
  simultaneously; the ranking helps but
  doesn't guarantee the right answer for
  ambiguous inputs.
- Database is a snapshot of upstream
  PyWhat regexes; brand-new ID formats may
  not be covered until the next release.
- No semantic verification — it'll happily
  call any 16-digit number a "credit card"
  even if the Luhn checksum fails (use a
  Luhn verifier for that).
