# Production AWS deployment for a SaaS

> **Anonymized case study.** No real hostnames, IPs, resource IDs, account
> numbers, secrets, or client details appear here. Code snippets are
> **illustrative** rewrites that show the *pattern* — not the actual production
> configuration.

## Context

A small business needed its web product — a **FastAPI + PostgreSQL 16 + React**
stock-management app — deployed as a live, low-cost, low-maintenance service with
its own domain and automatic HTTPS. Budget was tight (single small instance) and
the data had to survive redeploys and be backed up. I own the **DevOps /
infrastructure** track: the whole cloud estate, the deploy pipeline, and keeping
it running.

## Architecture

```
                          Internet
                             │  (HTTPS, automatic TLS)
                             ▼
            ┌──────────────────────────────────┐
            │  EC2 (t3.micro + swap)            │
            │                                   │
            │   Caddy  ──►  api (FastAPI)       │
            │   reverse     │                   │
            │   proxy       ▼                   │
            │             db (PostgreSQL 16)    │
            │               │                   │
            └───────────────┼───────────────────┘
                            │ data on
                            ▼
                 ┌────────────────────────┐
                 │ Persistent EBS volume  │  (survives instance replace)
                 └────────────────────────┘

   SSM Parameter Store ──► secrets injected at boot (never in code/metadata)
   S3 bucket           ◄── daily pg_dump backup (systemd timer, retention)
   IAM role            ──► instance access to SSM + S3 (least privilege)
   AWS Budgets         ──► cost alarm by email
```

Everything above is **Terraform** (~30 resources): VPC + subnet + security group,
the EC2 instance and its Elastic IP, the **separate persistent EBS volume** for
the database, the IAM role/policy, the S3 backups bucket with a lifecycle rule,
SSM SecureString parameters, and a budget alarm.

## Key decisions (and why)

| Decision | Why |
|---|---|
| **Database on a separate persistent EBS volume**, not the instance disk | Redeploying / replacing the instance never loses data. Compute is disposable; data is not. |
| **Secrets in SSM Parameter Store (SecureString)**, fetched at boot | Never in the repo, never in EC2 user-data/metadata. Rotatable without a code change. |
| **Caddy as reverse proxy** | Automatic Let's Encrypt TLS + HTTPS redirect with almost no config — right call for a one-box deploy. |
| **Daily `pg_dump` → S3 with retention** (systemd timer) | Cheap, restorable off-box backups; retention keeps cost bounded. |
| **Continuous deployment via a systemd timer** | The host polls `main`, pulls, and rebuilds — new commits go live with zero manual steps, no data loss. |
| **Least-privilege IAM + EU-region hosting** | Instance can reach only the SSM paths and the backup bucket it needs; EU hosting for GDPR. |

## My role

- Wrote the Terraform for the entire estate and the production container
  configuration (reverse proxy + app + db, no host-exposed database port).
- Built the backup and continuous-deployment automation (systemd timers).
- **Debugged real production incidents on the live system**, including:
  - a TLS/ACME misconfiguration that left the site without a certificate, and
  - an ORM enum mismatch that broke login with a 500 after a deploy —
    diagnosed from logs, fixed, and added a regression guard.

## Outcome

A service on a custom domain with automatic HTTPS, running at roughly the
cost of a single small instance, with **daily off-box backups** and
**zero-touch deploys**. Data survives instance replacement by design.

## Tech

`Terraform` · `AWS (EC2, EBS, VPC, IAM, SSM Parameter Store, S3, Budgets)` ·
`Docker Compose` · `Caddy` · `PostgreSQL 16` · `GitHub Actions` · `systemd`

---

### Illustrative snippet — secrets via SSM (pattern, not the real config)

Secrets are generated in Terraform and stored as SSM SecureStrings; the instance
reads them at boot instead of ever putting them in code or metadata:

```hcl
# Generate a strong secret and keep it in SSM (encrypted), not in the repo.
resource "random_password" "db" {
  length  = 32
  special = false
}

resource "aws_ssm_parameter" "db_password" {
  name  = "/app/prod/db_password"
  type  = "SecureString"
  value = random_password.db.result
}

# The instance's IAM role is scoped to just these parameters — nothing else.
data "aws_iam_policy_document" "ssm_read" {
  statement {
    actions   = ["ssm:GetParameter", "ssm:GetParameters"]
    resources = ["arn:aws:ssm:*:*:parameter/app/prod/*"]
  }
}
```

*(Values, names, and ARNs above are generic examples — the production estate
uses its own paths and least-privilege scoping.)*
