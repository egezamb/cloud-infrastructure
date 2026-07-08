# Cloud Infrastructure — Case Studies

Write-ups of real production infrastructure I've **designed and operated** — not
runnable client code, but the architecture, decisions, and my role, with all
sensitive details removed (no secrets, no real hosts/IDs, clients anonymized).

For runnable code demonstrating the same patterns, see my public repos linked below.

---

## Case studies

- **[Production AWS deployment for a SaaS](docs/aws-production-saas.md)** — a live
  FastAPI + PostgreSQL + React product on a single EC2 host, fully managed with
  Terraform (~30 resources): persistent-EBS Postgres, automatic-TLS reverse proxy,
  secrets in SSM, daily S3 backups, and a zero-touch continuous-deployment pipeline.
  *I own the DevOps/infrastructure track and debugged real production outages on it.*

- **[Kubernetes + Helm for a multi-service SaaS](docs/kubernetes-helm-saas.md)** —
  containerized several services (API, workers, web) into one Helm chart with
  per-environment values, a commit-SHA image pipeline, Ingress/TLS, and HPA.
  Deploy or update the whole platform with a single `helm upgrade --install`.

- **[CI/CD + secrets hardening for an app backend](docs/cicd-and-secrets-hardening.md)** —
  GitHub Actions CI on every change, safe continuous deployment on merge, and a
  security pass: AES-256-GCM encryption of access tokens at rest and CSPRNG
  session IDs.

---

## What these demonstrate

- **Infrastructure as Code** — Terraform for the full estate (VPC, compute, storage,
  IAM, secrets, backups, budget alarms), reproducible from `terraform apply`.
- **Secure by default** — secrets in SSM Parameter Store (never in code/metadata),
  least-privilege IAM, encrypted storage, restricted network access, GDPR-aware EU hosting.
- **Operations** — persistent data separated from disposable compute, automated
  backups with retention, continuous deployment, and hands-on incident debugging on
  live systems (TLS/ACME failures, an ORM enum regression).
- **CI/CD** — GitHub Actions gating every change (lint, test, build) before it ships.

## Runnable public code

| Repo | Demonstrates |
|---|---|
| [aws-terraform-starter](https://github.com/egezamb/aws-terraform-starter) | Structured Terraform: VPC + EC2 + S3, secure defaults, CI |
| [aws-lambda-terraform-lab](https://github.com/egezamb/aws-lambda-terraform-lab) | Serverless IaC: scheduled Lambda + API Gateway |

## About

Ege Zambelli — junior Cloud / DevOps engineer, Wrocław. ~2 years building AWS/Azure
infrastructure for paying clients. Open to junior roles.
📧 egezambelli@protonmail.com
