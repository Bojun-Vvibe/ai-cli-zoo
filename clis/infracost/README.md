# infracost

- **Repository:** https://github.com/infracost/infracost
- **Latest version:** v0.10.44
- **License:** Apache-2.0 — verified at [`LICENSE`](https://github.com/infracost/infracost/blob/master/LICENSE) (raw: https://raw.githubusercontent.com/infracost/infracost/master/LICENSE)
- **Niche:** Infrastructure-as-code cost estimation / FinOps in CI

## What it does

`infracost` reads Terraform / OpenTofu plans (or HCL directly) and
produces a **cloud bill estimate** for the resources that plan would
create, modify, or destroy. It supports AWS, Azure, and GCP pricing,
and outputs human-readable diffs, JSON, HTML, GitHub/GitLab PR
comments, and Slack-friendly summaries.

```
infracost breakdown --path .
infracost diff --path . --compare-to infracost-base.json
infracost comment github --pr 123 --path infracost.json
```

## Why interesting

The conventional CI gate for infrastructure changes is "does it plan?
does it apply?" That catches syntax and policy violations, but it
silently waves through the change that quietly turns a $40/month
NAT gateway into a $4,000/month one because someone scaled an
auto-scaling group's max size or flipped a storage class.

`infracost diff` puts a dollar number on every PR before merge, which
turns FinOps from a quarterly retro exercise into a per-change review
signal. For agent-driven infra automation specifically, that's the
difference between "the agent can ship Terraform PRs" and "the agent
can ship Terraform PRs and you can sleep at night".

## Pairs well with

- `conftest` / OPA — for *policy* gates that complement *cost* gates.
- `terragrunt` / `tofu` — for the actual IaC orchestration whose plans
  you're pricing.
- `dyff` — for diffing the rendered config that produced the cost
  delta when the number surprises you.

## Note

Pricing data is fetched from a hosted Cloud Pricing API; you need a
free API key (`infracost auth login`) for the public endpoint, or you
can self-host the pricing API for air-gapped use.
