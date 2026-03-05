# CI/CD Pipeline – Dev → Test → Prod (Cloudflare Pages)

This repository implements a multi-branch CI/CD pipeline using GitHub Actions with automated security gates, controlled promotion, production verification, and scheduled artifact backups.

Designed for structured deployments to Cloudflare Pages, this pipeline enforces progressive hardening from `dev` → `test` → `prod`.

---

## Branch Strategy

```
dev  →  test  →  prod
```

| Branch | Purpose | Security Level | Deployment |
|--------|----------|---------------|------------|
| `dev`  | Development & early validation | Moderate audit (non-blocking) | No |
| `test` | Pre-production validation | High audit (blocking) | Auto-merge to prod on success |
| `prod` | Production-ready code | High audit + final verification | Auto-deploy via Cloudflare |
| (scheduled) | Backup of prod artifacts | N/A | Artifact retention |

---

# Workflows

All workflows live under:

```
.github/workflows/
```

---

## 1️⃣ dev.yml – Development CI

**Triggers**
- Push to `dev`
- Pull request targeting `dev`

### What It Does

- Installs dependencies (`npm ci`)
- Checks outdated packages
- Runs a non-blocking security audit (`moderate` level)
- Builds the project

### Security Model

```
npm audit --omit=dev --audit-level=moderate
```

- Does NOT block merge
- Intended for early warning
- Encourages remediation before promotion

### Outcome

If successful:
- Build confirmation
- Suggestion to create PR to `test`

---

## 2️⃣ test.yml – Pre-Production Gate

**Triggers**
- Push to `test`
- Pull request targeting `test`

### What It Does

- Strict production dependency audit
- Blocks on `high` vulnerabilities
- Builds the project
- Runs preview
- Auto-merges `test` → `prod` if successful

### Security Gate

```
npm audit --omit=dev --audit-level=high
```

This step is BLOCKING.

If vulnerabilities are found:
- Merge is prevented
- `prod` remains untouched

### Auto-Merge Requirements

Requires a Personal Access Token stored as:

```
secrets.PAT_TOKEN
```

Used for pushing to `prod`.

---

## 3️⃣ prod.yml – Production Deployment

**Trigger**
- Push to `prod`

This is the final verification stage before Cloudflare deployment.

### What It Does

- Final security audit
- Production build with `NODE_ENV=production`
- Generates deployment metadata
- Creates build artifact (7-day retention)
- Blocks deployment on failure

### Security Enforcement

```
npm audit --omit=dev --audit-level=high
```

### Deployment Model

Cloudflare Pages is configured to:

```
Auto-deploy from the prod branch
```

GitHub verifies integrity before Cloudflare deploys.

---

## 4️⃣ backup.yml – Scheduled Artifact Backup

**Trigger**
- Every Sunday at 12:00 UTC (4 AM PST standard time)
- Manual trigger via Actions tab

### What It Does

- Checks out `prod`
- Builds project
- Archives `.svelte-kit/`
- Uploads artifact (30-day retention)

Artifact naming format:

```
backup-YYYYMMDD-HHMMSS-COMMIT_SHA.tar.gz
```

### Purpose

- Weekly production state snapshot
- Recovery mechanism
- Deployment traceability

---

# Security Model Overview

| Stage | Audit Level | Blocking |
|-------|------------|----------|
| dev   | moderate   | ❌ No |
| test  | high       | ✅ Yes |
| prod  | high       | ✅ Yes |

Only production dependencies are audited:

```
--omit=dev
```

This ensures runtime security integrity without noise from dev-only tooling.

---

# Artifacts

| Workflow | Artifact | Retention |
|----------|----------|-----------|
| backup   | Scheduled backup | 30 days |
| prod     | Production build | 7 days |

Artifacts are available in the **Actions → Workflow Run → Artifacts** section.

---

# Requirements

- Node.js 20
- `npm ci` compatible lockfile
- Cloudflare Pages configured to deploy from `prod`
- GitHub secret:
  ```
  PAT_TOKEN
  ```

---

# Deployment Flow Example

1. Push feature → `dev`
2. CI validates build + warns on moderate vulnerabilities
3. Merge `dev` → `test`
4. Strict audit runs
5. Auto-merge into `prod` if clean
6. Production verification runs
7. Cloudflare auto-deploys
8. Weekly backups stored automatically

---

# Design Philosophy

- Progressive security hardening
- Zero-trust promotion model
- Immutable production branch
- Automated recovery artifacts
- Deterministic builds via `npm ci`

---

# Observability

Production deployment metadata includes:

- Commit SHA
- Branch name
- Triggering user
- UTC timestamp

This improves auditability and rollback clarity.

---

# Failure Handling

| Failure Point | Effect |
|--------------|--------|
| Dev audit fails | Warning only |
| Test audit fails | Blocks promotion |
| Prod build fails | Blocks deployment |
| Backup fails | Logged in Actions |

---

# Directory Structure

```
.github/
└── workflows/
    ├── dev.yml
    ├── test.yml
    ├── prod.yml
    └── backup.yml
```

---

# License

MIT License

Copyright (c) 2026 Ricardo Godinez

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
