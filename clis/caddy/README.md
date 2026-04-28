# caddy

- **Repo**: https://github.com/caddyserver/caddy
- **Version**: v2.11.2
- **License**: Apache-2.0 (`LICENSE`)

## What it is

A Go-written HTTP/2 and HTTP/3 web server with **automatic HTTPS** as the default behavior: it provisions and renews TLS certificates from Let's Encrypt or ZeroSSL on demand, including for arbitrary client-supplied SNI hostnames via on-demand TLS. Configuration is driven by either the human-friendly `Caddyfile` or a JSON API, and the server is fully API-reconfigurable at runtime without restart.

## Why pick this over alternatives

Pick caddy over `nginx`/`traefik`/`httpd` when you want **HTTPS to just work without writing a single cert renewal cron or ACME hook**: a one-line `example.com { reverse_proxy localhost:8080 }` Caddyfile gets you a publicly-trusted, auto-renewing TLS site, which neither nginx nor httpd give you out of the box.

## Install

```sh
brew install caddy
```

## Quick example

```sh
caddy reverse-proxy --from example.com --to localhost:8080
# automatically obtains a Let's Encrypt cert for example.com
```
