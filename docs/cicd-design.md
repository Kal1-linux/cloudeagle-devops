# CI/CD Design Document — `sync-service`

**Service:** Spring Boot `sync-service`  
**Backend:** MongoDB  
**Target:** GCP VMs  
**Build System:** Jenkins  
**Environments:** `qa` → `staging` → `prod`

---

## 1. Branching Strategy

### Branch → Environment Mapping

```
feature/*  ──→  (local only, no deploy)
              ↓ PR
develop    ──→  QA (auto-deploy on merge)
              ↓ PR
staging    ──→  Staging (auto-deploy on merge)
              ↓ PR (manual approval required)
main       ──→  Production (deploy only via tagged release)
```

| Branch | Environment | Deploy Trigger | Approval Required |
|--------|-------------|----------------|-------------------|
| `develop` | QA | Merge to `develop` | No |
| `staging` | Staging | Merge to `staging` | No |
| `main` | Production | Git tag `v*.*.*` only | Yes — 2 approvers |

### Guarding Against Accidental Prod Deployments

1. **Branch protection on `main`:** No direct pushes; only merges via PR with 2 approvals.
2. **Tag-gated deployment:** The prod Jenkins job only triggers on tags matching `v[0-9]+.[0-9]+.[0-9]+`. A merge to `main` alone does NOT deploy.
3. **Environment lock:** The `prod` Jenkins job requires a manual "Proceed" input step after the pre-flight health check.
4. **Separation of duties:** The person who merges the PR cannot be one of the two required approvers.

---

## 2. Jenkins Pipeline

### High-Level Stages

```
Checkout → Test → Build → Docker Build/Push → Deploy → Smoke Test → Notify
```

| Stage | QA | Staging | Prod |
|-------|----|---------|------|
| Checkout | ✓ | ✓ | ✓ |
| Unit Tests | ✓ | ✓ | ✓ |
| Integration Tests | ✓ | ✓ | ✓ |
| SAST (SpotBugs / OWASP) | ✓ | ✓ | ✓ |
| Docker Build & Push | ✓ | ✓ | ✓ |
| Deploy | auto | auto | manual gate |
| Smoke Test | ✓ | ✓ | ✓ |
| Load Test (k6) | — | ✓ | — |
| Manual Approval | — | — | ✓ |

### PR vs Merge Behaviour

**On PR (no merge yet):**
- Checkout, compile, unit tests, integration tests, SAST scan.
- Docker image is **not** built; no deployment.
- Results posted as GitHub/GitLab status checks — PR cannot merge if any fail.

**On Merge:**
- Full pipeline runs including Docker build, push to Artifact Registry, and environment deploy.
- For `prod`, the pipeline pauses at the manual approval gate.

### Rollback Strategy

**Automated rollback** triggers if the smoke test fails post-deployment:

```
Deploy new version
    ↓
Smoke test (3 min window, 5 retries)
    ↓ FAIL
Rollback: re-deploy previous Docker tag (stored as `PREVIOUS_IMAGE_TAG` in Jenkins env)
    ↓
Alert: PagerDuty / Slack #incidents
```

The previous stable image tag is stored as a Jenkins parameter `LAST_STABLE_TAG` updated only on successful deployments. A manual rollback job (`sync-service-rollback`) also exists for operator-initiated rollbacks.

---

## 3. Configuration Management

### Env-Specific Configs

Spring Boot profiles map 1:1 to environments:

```
src/main/resources/
  application.yml            # shared defaults
  application-qa.yml         # QA overrides
  application-staging.yml    # Staging overrides
  application-prod.yml       # Prod overrides (no secrets — placeholders only)
```

At runtime, the JVM flag `-Dspring.profiles.active=${ENV}` is injected via the VM's startup script (managed by Ansible / GCP metadata). Env-specific non-secret config (log levels, feature flags, replica counts) lives in the respective `application-{env}.yml` and is committed to the repo.

### Secrets Handling

**Tool: GCP Secret Manager**

MongoDB credentials, API keys, and any sensitive values are stored in GCP Secret Manager, **never** in source control or Jenkins credentials store for production.

```
projects/cloudeagle-prod/secrets/
  sync-service-mongo-uri
  sync-service-api-key
  sync-service-jwt-secret
```

At deploy time, the startup script (or a Kubernetes init container equivalent) fetches secrets:

```bash
MONGO_URI=$(gcloud secrets versions access latest \
  --secret="sync-service-mongo-uri" \
  --project="cloudeagle-prod")
export SPRING_DATA_MONGODB_URI="${MONGO_URI}"
```

The GCP VM's service account has `roles/secretmanager.secretAccessor` scoped only to its own env's secrets.

For Jenkins itself, build-time secrets (e.g., Artifact Registry push credentials, Sonar token) are stored as Jenkins Credentials (type: Secret text) and injected via `withCredentials` blocks — never echoed to logs.

---

## 4. Deployment Strategy

### Chosen Strategy: Blue/Green per Environment

**Why Blue/Green over Rolling or Recreate?**

| Strategy | Downtime | Rollback Speed | Resource Cost | Chosen? |
|----------|----------|----------------|---------------|---------|
| Recreate | Yes | Fast | Low | No |
| Rolling | Minimal | Slow (re-roll) | Low | No |
| Blue/Green | Zero | Instant | Medium | **Yes** |

**Justification:**  
`sync-service` connects to MongoDB and performs data synchronization — running two different schema versions simultaneously (as in rolling) risks data inconsistency. Blue/Green avoids this: traffic switches atomically at the load balancer once the green environment passes health checks.

### Zero-Downtime Flow

```
Current state: LB → Blue (v1.2.0)

1. Provision Green VMs with v1.3.0 (behind LB, no traffic)
2. Run smoke tests against Green directly via internal URL
3. Shift LB: 10% → Green (canary validation, 5 min)
4. Shift LB: 100% → Green
5. Keep Blue alive for 15 min (instant rollback window)
6. Decommission Blue
```

The LB traffic shift uses GCP's backend service weights via `gcloud compute backend-services` update commands in the Jenkinsfile, requiring zero VM restarts.

---

## 5. Jenkinsfile Reference

See `Jenkinsfile` at the repo root for the full implementation.

Key design choices in the Jenkinsfile:
- Uses declarative pipeline with `agent { label 'gcp-builder' }`
- Environment variables injected via `withCredentials` and GCP Secret Manager at deploy time
- `DEPLOY_ENV` is derived from branch name automatically
- Prod deploy stage has `input` step gated by `Deployment Approvers` role
- `post { failure { ... } }` block triggers automated rollback
