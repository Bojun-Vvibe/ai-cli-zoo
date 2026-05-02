# octodns

> **DNS-as-code at fleet scale — a Python framework + CLI
> (`octodns-sync`) that treats DNS zones as version-controlled
> YAML files (`config/example.com.yaml` per zone, one record
> per key) and synchronises the desired state to *N* providers
> in parallel: Route 53, Cloudflare, Google Cloud DNS, Azure
> DNS, NS1, DNSimple, DigitalOcean, Hetzner, PowerDNS, BIND
> zonefiles, and ~30 more via per-provider plugin packages
> (`octodns-route53`, `octodns-cloudflare`, ...). Run
> `octodns-sync --config-file=config.yaml` for a dry-run plan
> (`+ A example.com 300 198.51.100.1` style diff per zone per
> provider), `--doit` to apply, `--force` to override safety
> checks; multi-provider per zone gives **DNS-level failover by
> committing the same zone to two providers and round-robin
> NS-ing**, and the `MetaProcessor` plugin layer rewrites records
> in-flight (geo-routing, weighted policies, dynamic record
> generation from inventory).** Pinned to **v1.16.0** (April
> 2025),
> [LICENSE](https://github.com/octodns/octodns/blob/main/LICENSE),
> MIT.

Source: <https://github.com/octodns/octodns>

## TL;DR

`octodns` is the **terraform of DNS**: a single declarative
representation (per-zone YAML), an idempotent planner that
diffs desired vs live state, and a multi-provider applier that
keeps two or three DNS hosts in lockstep so a registrar-level
NS swap is the disaster-recovery story instead of a
3 a.m. zonefile re-import. Originally built and run inside
GitHub to manage `github.com` and friends across multiple
providers, now community-maintained under the `octodns`
GitHub org. The core (`octodns`) ships only the engine and the
config / plan / sync machinery; provider plugins are *separate*
PyPI packages (`pip install octodns-route53 octodns-cloudflare`)
so the install footprint scales to what you actually use.
Records are typed (`A`, `AAAA`, `ALIAS`, `CAA`, `CNAME`, `DS`,
`LOC`, `MX`, `NAPTR`, `NS`, `PTR`, `SPF`, `SRV`, `SSHFP`, `TLSA`,
`TXT`, `URLFWD`, plus `dynamic` for weighted / geo records),
TTLs are explicit, and a YAML linter + `octodns-validate` keep
malformed zones out of CI.

## Install

```bash
# Pip (recommended — install the engine plus only the providers you use)
pip install octodns octodns-route53 octodns-cloudflare \
            octodns-bind octodns-powerdns

# Verify
octodns-sync --version    # octoDNS 1.16.0

# Project layout
mkdir -p octodns-prod/{config,zones,envs}
cd octodns-prod
```

## One Concrete Example

```yaml
# config.yaml — top-level orchestration
manager:
  max_workers: 4

providers:
  config:
    class: octodns.provider.yaml.YamlProvider
    directory: ./zones
    default_ttl: 300
    enforce_order: true

  route53:
    class: octodns_route53.Route53Provider
    access_key_id: env/AWS_ACCESS_KEY_ID
    secret_access_key: env/AWS_SECRET_ACCESS_KEY

  cloudflare:
    class: octodns_cloudflare.CloudflareProvider
    token: env/CLOUDFLARE_TOKEN

zones:
  example.com.:
    sources: [config]
    targets: [route53, cloudflare]   # two providers in lockstep
```

```yaml
# zones/example.com.yaml — the zone itself
'':
  - type: A
    ttl: 300
    values: [198.51.100.1, 198.51.100.2]
  - type: MX
    ttl: 3600
    values:
      - exchange: mx1.example.com.
        preference: 10
      - exchange: mx2.example.com.
        preference: 20
  - type: TXT
    ttl: 3600
    values:
      - 'v=spf1 include:_spf.example.com -all'
      - 'google-site-verification=abcdef'

www:
  - type: CNAME
    ttl: 300
    value: example.com.

api:
  - type: A
    ttl: 60
    values: [198.51.100.10]

_dmarc:
  - type: TXT
    ttl: 3600
    value: 'v=DMARC1; p=reject; rua=mailto:dmarc@example.com'
```

```bash
# Dry-run plan — prints the diff against EVERY target provider
octodns-sync --config-file=config.yaml

# *********** route53 zone: example.com. ************
# * Create
# *   - <ARecord A 300, '', [198.51.100.1, 198.51.100.2]>
# *   - <CNAMERecord CNAME 300, 'www', example.com.>
# * Summary: Creates=12, Updates=0, Deletes=0, Existing=0
# *
# * Plan
# *   Zone     Create  Update  Delete  Total
# *   example. 12      0       0       12
# ****************************************************

# Apply for real
octodns-sync --config-file=config.yaml --doit

# Validate before plan (catch typos, unsupported record types)
octodns-validate --config-file=config.yaml
```

CI integration (one of the canonical shapes):

```yaml
# .github/workflows/dns.yml
on: { pull_request: { paths: [zones/**, config.yaml] } }
jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install octodns octodns-route53 octodns-cloudflare
      - run: octodns-validate --config-file=config.yaml
      - run: octodns-sync   --config-file=config.yaml > plan.txt
      - uses: actions/upload-artifact@v4
        with: { name: dns-plan, path: plan.txt }
```

## License

[MIT](https://github.com/octodns/octodns/blob/main/LICENSE),
SPDX `MIT`.

## Niche / positioning

Pick `octodns` over [`dnscontrol`](../dnscontrol/) when the team
is Python-shaped, when the multi-provider lockstep story matters
(octodns was *built* for "the same zone in Route 53 *and*
Cloudflare so a single-provider outage does not take us down"),
or when the plugin ecosystem covers a niche provider you need
(octodns has ~40 providers via separate `octodns-*` packages;
dnscontrol bundles its providers in-tree). Pick `dnscontrol`
when the team prefers a JavaScript/DSL config (`D("example.com",
REG_NAMECOM, ...)`) and a single static Go binary install with
no Python toolchain. Pick over raw [`terraform`](../terraform/)
+ provider blocks when DNS specifically is the workload —
terraform models DNS records as a side-effect of resource
graphs and the diff output is generic; octodns-sync gives a
DNS-shaped plan (record-level adds / changes / deletes per
provider) that an on-call engineer can read at 2 a.m. without
mental translation. Pair with [`dnscontrol`](../dnscontrol/)
*only* as the alternative-pick branch (do not run both against
the same zones), with [`dig`](../dig/) / [`drill`](../drill/) /
[`dog`](../dog/) for post-apply verification, and with
[`bind`](https://www.isc.org/bind/) zonefile import via the
`octodns-bind` provider when migrating off self-hosted DNS.
Skip for a single small zone (one A record + a couple of MX
entries) on a single provider — the provider's own console or
[`cli53`](https://github.com/barnybug/cli53) /
[`flarectl`](https://github.com/cloudflare/cloudflare-go) is
lighter weight; octodns earns its keep when zones × providers
× records crosses a few dozen.
