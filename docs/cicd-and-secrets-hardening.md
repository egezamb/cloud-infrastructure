# CI/CD pipeline + secrets hardening for an app backend

> **Anonymized case study.** No product, provider, host, or secret appears here.
> Snippets are **illustrative** patterns.

## Context

A backend service (Node + TypeScript) needed two things: an automated safety net
so nothing broken reaches `main`, and safe **continuous deployment** — plus a
security pass on how it persisted third-party **access tokens** and issued
**session identifiers**. I built the CI/CD and did the hardening.

## What I built

### 1. CI on every change
GitHub Actions runs on every pull request and push to `main`:
`typecheck → lint → test → build`. Nothing merges red. This caught regressions
before they shipped and made reviews about logic, not formatting.

### 2. Continuous deployment on merge
Merging to `main` triggers a deploy job that builds and ships the new version to
the container host. Two guardrails matter:
- **Skip-safe:** if the deploy credential isn't configured yet, the job **warns
  and skips** instead of failing red — so the pipeline never blocks the team
  before infra is wired up.
- **Concurrency + path filters:** deploys never overlap, and only deploy-relevant
  changes trigger one.

### 3. Secrets hardening
- **Access tokens encrypted at rest** — long-lived third-party tokens were being
  stored in plaintext. I added transparent **AES-256-GCM** encryption at the
  persistence boundary (authenticated, versioned, rotatable), so the rest of the
  app is unchanged and the database never holds plaintext tokens.
- **Unguessable session IDs** — session identifiers (the bearer credential
  guarding a live token) were generated with a non-cryptographic RNG. I switched
  to a **CSPRNG** (256-bit), and set the session cookie to `Secure` + `HttpOnly`
  in production.

## Key decisions (and why)

| Decision | Why |
|---|---|
| **CI gates before merge** | A broken `main` blocks everyone; cheap checks prevent expensive incidents. |
| **Deploy skips (not fails) without its secret** | The pipeline is safe to merge before Ops finishes wiring credentials. |
| **Encrypt tokens at the persistence layer** | Transparent to all callers; legacy plaintext rows re-encrypt on next write — no migration, no risk. |
| **CSPRNG for session IDs** | The ID protects a live access token; guessable IDs = account takeover. |

## Outcome

Every change is tested before it lands; merges deploy themselves safely; and the
stored access tokens and session IDs are no longer a plaintext / guessable
liability. All changes shipped through reviewed PRs with green CI.

## Tech

`GitHub Actions` · `Docker` · `Node + TypeScript` · `AES-256-GCM (node:crypto)` ·
`CSPRNG` · `Continuous Deployment`

---

### Illustrative snippet — encrypt-at-rest (pattern, not the real module)

Authenticated, versioned, and self-describing so it's rotatable:

```ts
import { createCipheriv, createDecipheriv, randomBytes } from "crypto";

// Stored form: v1:<iv>:<authTag>:<ciphertext> (all base64url)
export function encrypt(plaintext: string, key: Buffer): string {
  const iv = randomBytes(12);
  const c = createCipheriv("aes-256-gcm", key, iv);
  const ct = Buffer.concat([c.update(plaintext, "utf8"), c.final()]);
  const tag = c.getAuthTag();
  return ["v1", iv, tag, ct].map((p) =>
    Buffer.isBuffer(p) ? p.toString("base64url") : p).join(":");
}

// Legacy plaintext (no "v1:" prefix) passes through — no migration needed.
export function decrypt(stored: string, key: Buffer): string {
  if (!stored.startsWith("v1:")) return stored;
  const [, iv, tag, ct] = stored.split(":").map((p, i) =>
    i === 0 ? p : Buffer.from(p, "base64url")) as [string, Buffer, Buffer, Buffer];
  const d = createDecipheriv("aes-256-gcm", key, iv);
  d.setAuthTag(tag);
  return Buffer.concat([d.update(ct), d.final()]).toString("utf8");
}
```

*(Illustrative — the production module has its own key management and tests.)*
