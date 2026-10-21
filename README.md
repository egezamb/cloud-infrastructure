# Cloud Infrastructure — Case Studies

Write-ups of real production infrastructure I've **designed and operated** — not
runnable client code, but the architecture, decisions, and my role, with all
sensitive details removed (no secrets, no real hosts/IDs, clients anonymized).

For runnable code demonstrating the same patterns, see my public repos linked below.

---

## Case studies

- **[Production AWS deployment for a SaaS](docs/aws-production-saas.md)** — a
  FastAPI + PostgreSQL + React product on a single EC2 host, fully managed with
  Terraform (~30 resources): persistent-EBS Postgres, automatic-TLS reverse proxy,
  secrets in SSM, daily S3 backups, and a zero-touch continuous-deployment pipeline.
  *I own the DevOps/infrastructure track and debugged real production outages on it.*

- **[Kubernetes + Helm for a multi-service SaaS](docs/kubernetes-helm-saas.md)** —
  containerized the services (API, Celery worker, web, PostgreSQL, Redis) into one
  Helm chart, with a CI pipeline that builds, scans and pushes commit-SHA-tagged
  images and an Ingress that routes the API and the web app.
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
| [prod-saas-aws](https://github.com/egezamb/prod-saas-aws) | Production-style Terraform: ECS Fargate, RDS, ALB, alarms, `terraform test`, OIDC deploys |
| [openarchiver-helm](https://github.com/egezamb/openarchiver-helm) | Helm chart published as a signed OCI artifact, with install and upgrade tests in CI |
| [helmbulance](https://github.com/egezamb/helmbulance) | Go CLI that tests Helm charts on a throwaway kind cluster (co-created) |

## About

Ege Zambelli — junior Cloud / DevOps engineer, Wrocław. Freelance AWS/Azure work
for small-business clients since 2023. Open to junior roles.
📧 egezambelli1@gmail.com
