# dnscontrol

> **Infrastructure-as-code for DNS** — declare every zone you own
> in a single JavaScript-flavoured DSL (`dnsconfig.js`), then push
> the diff to 35+ DNS providers (Route 53, Cloudflare, Google, Azure
> DNS, NS1, DigitalOcean, Hetzner, Hurricane Electric, Namecheap,
> Gandi, Linode, etc.) with one command. Pinned to **v4.36.1**
> ([LICENSE](https://github.com/itstoragesvc/dnscontrol/blob/main/LICENSE),
> MIT).

Source: <https://github.com/StackExchange/dnscontrol>

(Note: the canonical GitHub URL is `StackExchange/dnscontrol`; the
`itstoragesvc` org owns the latest release tag at snapshot but the
project is the same Stack Exchange-originated tool.)

## TL;DR

DNS zones rot when N people make ad-hoc edits in N different
provider portals. dnscontrol forces the entire DNS surface into
one versioned `dnsconfig.js` file in git, with a `creds.json`
listing per-provider API tokens. `dnscontrol preview` shows a
human-readable diff of *every* record that would change against
*every* configured provider; `dnscontrol push` applies it. The DSL
is real JavaScript so you get loops, helper functions, and macros
for the "10 subdomains all pointing at the same load balancer"
pattern instead of 10 copy-pasted lines. The diff is computed by
fetching the live zone from each provider's API and comparing it
to your declaration — drift detection is automatic, and `push`
will *remove* records that exist live but not in code (with an
explicit `IGNORE` escape hatch for records managed elsewhere).
Multi-provider is first-class: a zone served by both Route 53 and
Cloudflare for redundancy is one declaration with two `DnsProvider`
calls, and the same record set goes to both.

## Install

```bash
# Homebrew (macOS / Linux)
brew install dnscontrol

# Go install
go install github.com/StackExchange/dnscontrol/v4@v4.36.1

# Pre-built binary
curl -L https://github.com/StackExchange/dnscontrol/releases/download/v4.36.1/dnscontrol-Darwin-arm64.tar.gz | tar xz

# Docker
docker run --rm -v "$PWD:/dns" stackexchange/dnscontrol preview
```

## Example

```javascript
// dnsconfig.js
var REG_NONE = NewRegistrar("none");
var DSP_CF = NewDnsProvider("cloudflare");

D("example.com", REG_NONE, DnsProvider(DSP_CF),
  A("@", "203.0.113.10"),
  A("www", "203.0.113.10"),
  CNAME("docs", "example.github.io."),
  MX("@", 10, "mail.example.com."),
  TXT("@", "v=spf1 include:_spf.google.com ~all"),
);
```

```bash
# Preview the diff without applying
dnscontrol preview

# Apply
dnscontrol push

# Validate the JS without hitting any provider
dnscontrol check
```

## When to use

- You own more than ~3 zones, or one zone with more than ~30
  records, and tracking changes across the team has become a Slack
  thread of "who added that CNAME".
- You want PR-reviewable DNS changes with a real diff, the same
  way you review Terraform / Helm changes.
- You need to keep the same zone in sync across two providers (DNS
  redundancy) and refuse to maintain that by hand.

## When NOT to use

- You have one zone with five records that never change — a
  provider portal is fine; dnscontrol is overhead.
- You need DNSSEC key management or registrar transfers — those
  remain provider-side, dnscontrol manages records not registry
  state.
- Your provider is not in the supported list (check the upstream
  README) and you do not want to write the provider plugin yourself.
