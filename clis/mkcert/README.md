# mkcert

> **Zero-config CLI that creates a local Certificate Authority,
> installs its root cert into the system + browser trust stores,
> and issues per-hostname leaf certs trusted by every browser on
> the machine — so `https://localhost` / `https://*.test` /
> `https://my-dev-host` Just Works without `--insecure` flags or
> `Not Secure` browser warnings.** Pinned to **v1.4.4**, BSD-3-Clause
> ([LICENSE](https://github.com/FiloSottile/mkcert/blob/master/LICENSE)).

- **Repo:** https://github.com/FiloSottile/mkcert
- **Latest version:** v1.4.4 (2022-04-26)
- **License:** BSD-3-Clause (`LICENSE` at repo root, SPDX
  `BSD-3-Clause`)
- **Category:** `tls` / `dev-loop` / `pki`
- **Language:** Go

## What it does

`mkcert -install` (run once per machine) generates a
local-only root CA in `$(mkcert -CAROOT)`
(`~/Library/Application Support/mkcert/` on macOS,
`~/.local/share/mkcert/` on Linux) and installs the public root
cert into the system trust store, the Firefox / Chrome / Java
trust stores it can find, and `nss` if present. After that,
`mkcert localhost 127.0.0.1 ::1 example.test "*.example.test"`
emits two files in the cwd
(`localhost+4.pem` + `localhost+4-key.pem`) that any local web
server, reverse proxy, or `curl` will accept under HTTPS
without `--insecure`, because the issuing CA is now trusted by
the OS / browser. Supports IP SAN, wildcard SAN,
ECDSA-P256 (`-ecdsa`), client auth certs (`-client`), PKCS#12
bundles (`-pkcs12` for Java / .NET / Windows IIS),
and uninstall (`mkcert -uninstall` removes the root CA
cleanly). Single Go binary, no daemon, no config file.

## Why included

The "I need real HTTPS for local development because the
browser refuses to test a `Secure` cookie / Service Worker /
HTTP/2 / WebAuthn / mixed-content path over plain HTTP"
problem used to mean either painful manual OpenSSL +
`security add-trusted-cert` choreography or shipping
`-k` / `--insecure` / `NODE_TLS_REJECT_UNAUTHORIZED=0` hacks
that mask real cert bugs. `mkcert` reduces it to two
commands: `mkcert -install` once, then `mkcert myhost.test`
per project. For an LLM-CLI workflow that spins up a local
web service the agent then drives a headless browser
against, generating the cert with `mkcert` (instead of
self-signing + tolerating warnings) lets the browser
automation run with default trust settings — which more
faithfully mirrors production-CA behavior than
`--ignore-certificate-errors` does. Hard caveat: the root
CA is *machine-local*, never push it to a shared host; for
team-wide trust use a real CA (Smallstep `step-ca`,
Let's Encrypt internal, etc.).
