# Security Scan Reports

Automated security scan findings from scheduled CronJobs running on both GCP and Azure clusters.

## How It Works

CronJobs run on a schedule using the [security-tools](https://github.com/mikelear/leartech-dockerfiles) BlackArch image. When findings are detected, GitHub Issues are automatically created in this repo with severity labels and cluster tags.

## Scanners

| Scanner | What it does | Schedule (Production) | Schedule (Staging) |
|---------|-------------|----------------------|-------------------|
| **Nuclei** | Dynamic vulnerability scanning against live endpoints | Weekly (Sunday 1am UTC) | Nightly (1am UTC) |
| **Nikto** | Web server misconfiguration detection | Weekly (Sunday 3am UTC) | Weekly (Sunday 3am UTC) |
| **Nmap** | Port exposure verification (unexpected open ports) | Weekly (Sunday 4am UTC) | Weekly (Sunday 4am UTC) |
| **Grype** | CVE scanning of our container images (including the scanner itself) | Daily (6am UTC) | Daily (6am UTC) |

## Labels

| Label | Meaning |
|-------|---------|
| `critical` / `urgent` | Requires immediate attention (4h SLA) |
| `high` | Next business day (24h SLA) |
| `medium` | Within 1 week |
| `low` | Next sprint |
| `gcp` / `az` | Which cluster detected the finding |
| `base-image-compromise` | CVE found in our BlackArch base image |
| `false-positive` | Investigated and confirmed not a real issue |
| `resolved` | Finding has been remediated |

## Response Process

See [RUNBOOK.md](RUNBOOK.md) for the full incident response playbook.

## Trust Chain

```
BlackArch base image
  → Pre-build Grype gate (fail on critical CVEs)
    → Kaniko build
      → Cosign sign
        → Kyverno verify at admission (Enforce mode)
          → Daily Grype re-scan of running images
```
