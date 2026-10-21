# Kubernetes + Helm for a multi-service SaaS

> **Anonymized case study.** No client, product name, cluster, registry, domain,
> or secret appears here. Snippets are **illustrative** patterns, not the real
> chart.

## Context

A B2B SaaS platform made of several services (an API, a Celery worker, a web
frontend, PostgreSQL and Redis) needed **reproducible deployments** instead of
hand-edited manifests. I own
the **DevOps track**: containerizing the services, packaging them as a **Helm
chart**, and wiring the image build pipeline and CI.

## Architecture

```
   Git push ──► CI (GitHub Actions)
                 │  build + test + scan
                 │  docker build → push (tagged by commit SHA)
                 ▼
          Container registry
                 │
                 ▼
      Helm chart  ──►  Kubernetes cluster
       (one chart,         ├─ Deployment: api
        values.yaml)       ├─ Deployment: worker
                           ├─ Deployment: web
                           ├─ Deployment: redis
                           ├─ StatefulSet: postgres
                           ├─ Services + Ingress (/api → api, / → web)
                           ├─ ConfigMap (non-secret config)
                           └─ Secret (injected env)
```

One Helm chart templates every service; `values.yaml` supplies image tags,
replica counts, resource limits, and the hostname.
Deploying the platform is one `helm upgrade --install`.

## Key decisions (and why)

| Decision | Why |
|---|---|
| **One Helm chart for every service** | All services deploy from the *same* templates and helpers — no copy-pasted YAML. |
| **Images tagged by commit SHA** (not `latest`) | Every deploy is traceable to an exact commit; rollbacks are deterministic. |
| **Resource requests/limits from values** | Requests/limits keep one service from starving the node. |
| **Config split: ConfigMap vs Secret** | Non-secret settings are visible/reviewable; secrets are injected separately and never committed. |
| **One Ingress for the platform** | Single entry point: `/api` goes to the API, everything else to the web app. |

## My role

- Wrote the Helm chart — templates, `_helpers.tpl` and values.
- Built the **Docker image pipeline** (commit-SHA tags, image scan before push)
  and the GitHub Actions CI that produces and pushes them.
- Set up the Deployment/StatefulSet/Service/Ingress topology and the
  ConfigMap/Secret separation.

## Outcome

A single `helm upgrade --install` deploys or updates the whole platform from
images built and tagged automatically by CI — repeatable and traceable.

## Tech

`Kubernetes` · `Helm` · `Docker (multi-stage)` · `GitHub Actions` ·
`Ingress`

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
