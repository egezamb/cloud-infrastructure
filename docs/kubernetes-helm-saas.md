# Kubernetes + Helm for a multi-service SaaS

> **Anonymized case study.** No client, product name, cluster, registry, domain,
> or secret appears here. Snippets are **illustrative** patterns, not the real
> chart.

## Context

A B2B SaaS platform made of several services (an API, background workers, and a
web frontend) needed **reproducible deployments** that behaved the same across
environments — no more hand-edited manifests drifting between dev and prod. I own
the **DevOps track**: containerizing the services, packaging them as a **Helm
chart**, and wiring the image build pipeline and CI.

## Architecture

```
   Git push ──► CI (GitHub Actions)
                 │  build + test
                 │  docker build → push (tagged by commit)
                 ▼
          Container registry
                 │
                 ▼
      Helm chart  ──►  Kubernetes cluster
        (one chart,        ├─ Deployment: api        (+ HPA)
         per-env values)   ├─ Deployment: worker
                           ├─ Deployment: web
                           ├─ Service + Ingress (TLS)
                           ├─ ConfigMap (non-secret config)
                           └─ Secret (injected env)
```

One Helm chart templates every service; a small `values-<env>.yaml` per
environment supplies image tags, replica counts, resource limits, and hostnames.
Deploying an environment is one `helm upgrade --install`.

## Key decisions (and why)

| Decision | Why |
|---|---|
| **One Helm chart, per-environment values files** | Dev and prod deploy from the *same* templates — environment parity, no drift, no copy-pasted YAML. |
| **Images tagged by commit SHA** (not `latest`) | Every deploy is traceable to an exact commit; rollbacks are deterministic. |
| **Resource requests/limits + HPA on the API** | The API scales on load; requests/limits keep one service from starving the node. |
| **Config split: ConfigMap vs Secret** | Non-secret settings are visible/reviewable; secrets are injected separately and never committed. |
| **Ingress with TLS termination** | Single managed entry point, HTTPS by default, host-based routing to services. |

## My role

- Wrote the Helm chart from scratch — templates, `_helpers.tpl`, values schema,
  and per-environment overrides.
- Built the **Docker image pipeline** (multi-stage builds, commit-SHA tags) and
  the GitHub Actions CI that produces and pushes them.
- Set up the Deployment/Service/Ingress/HPA topology and the ConfigMap/Secret
  separation.

## Outcome

A single `helm upgrade --install` deploys or updates the whole platform for a
given environment, from images built and tagged automatically by CI — repeatable,
traceable, and identical across environments.

## Tech

`Kubernetes` · `Helm` · `Docker (multi-stage)` · `GitHub Actions` ·
`Ingress / TLS` · `HPA`

---

### Illustrative snippet — templated Deployment (pattern, not the real chart)

A single template renders every service; values decide image, replicas, and limits:

```yaml
# templates/deployment.yaml (simplified)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}-{{ .Values.service.name }}
spec:
  replicas: {{ .Values.service.replicas }}
  template:
    spec:
      containers:
        - name: {{ .Values.service.name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits:   { cpu: "500m", memory: "512Mi" }
          envFrom:
            - configMapRef: { name: {{ include "app.fullname" . }}-config }
            - secretRef:    { name: {{ include "app.fullname" . }}-secret }
```

*(Names, values, and limits are generic examples.)*
