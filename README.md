# CloudEagle DevOps Assignment — sync-service

Spring Boot + MongoDB service · GCP · Jenkins CI/CD

## Repository Structure

```
.
├── Jenkinsfile                        # Declarative pipeline (Part 1)
├── docs/
│   ├── cicd-design.md                 # CI/CD design document (Part 1)
│   └── infrastructure-design.md      # Infrastructure design document (Part 2)
└── infra/
    └── architecture-diagram.svg      # GCP architecture diagram (Part 2)
```

## Part 1 — CI/CD Design

**[Design document →](docs/cicd-design.md)**  
**[Jenkinsfile →](Jenkinsfile)**

### Summary of decisions

| Topic | Decision |
|-------|----------|
| Branching | `feature → develop → staging → main`; tag-gated prod |
| Build | Jenkins declarative pipeline, `gcp-builder` agent |
| Docker registry | GCP Artifact Registry (asia-south1) |
| Deployment | **Blue/Green** via GCP Managed Instance Groups |
| Rollback | Automatic on smoke-test failure; `LAST_STABLE_TAG` stored in GCS |
| Secrets | GCP Secret Manager; Jenkins Credentials for build-time tokens |
| Config | Spring profiles (`qa` / `staging` / `prod`); no secrets in YAML |

### Pipeline stages

```
Checkout → Unit Tests → Integration Tests → SAST (SpotBugs + OWASP + Sonar)
→ Build JAR → Docker Build/Push → Deploy (Blue/Green) → Smoke Test
```

## Part 2 — Infrastructure Design

**[Design document →](docs/infrastructure-design.md)**  
**[Architecture diagram →](infra/architecture-diagram.svg)**

### Summary of decisions

| Topic | Decision | Rationale |
|-------|----------|-----------|
| Compute | **GKE Autopilot** | Per-pod billing, zero node ops, built-in HPA |
| MongoDB | **Atlas M10** (prod), M0 (qa/staging) | No ops, native GCP VPC peering |
| Ingress | Cloud Armor → HTTPS LB → NGINX | WAF + managed TLS at zero extra ops |
| Secrets | GCP Secret Manager + CSI driver | Zero secrets in cluster YAML |
| IAM | Workload Identity | No key files inside pods |
| Monitoring | Cloud Monitoring + Logging + Trace | Native GKE integration |
| Estimated cost | ~$120–170/mo (prod) | Scales with load, no idle waste |
