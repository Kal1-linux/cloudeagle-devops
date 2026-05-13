# Infrastructure Design — `sync-service` on GCP

**Constraint:** Startup budget — cost-conscious but production-grade  
**Requirements:** Auto-scaling · Secure access · MongoDB hosting

---

## 1. Compute Choice

### Decision: GKE Autopilot (not Compute Engine, not Cloud Run)

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **Compute Engine VMs** | Simple, full control | Manual scaling, patching, LB config | ❌ Too much ops overhead |
| **Cloud Run** | Serverless, zero-cost at idle | Cold starts hurt sync workloads; stateful connections tricky with MongoDB | ❌ Wrong fit for sync service |
| **GKE Standard** | Full Kubernetes | You manage node pools, sizing, upgrades | ❌ Expensive on small team |
| **GKE Autopilot** ✓ | No node management, per-pod billing, built-in HPA/VPA | Slightly less control than Standard | ✅ Best balance |

**Why GKE Autopilot:**
- Autopilot bills per pod resource request, not per node — idle capacity costs nothing.
- Built-in HPA (Horizontal Pod Autoscaler) handles traffic bursts automatically.
- Google manages control plane, node upgrades, and security patches.
- Spring Boot + MongoDB connection pooling works natively in Kubernetes.

---

## 2. MongoDB Hosting

### Decision: MongoDB Atlas (GCP-hosted, same region) — Free → M10 tier path

**Options considered:**

| Option | Cost | Ops Effort | HA | Verdict |
|--------|------|------------|-----|---------|
| Self-managed on GCE | Low | Very High | Manual | ❌ |
| MongoDB Atlas M10 | ~$57/mo | Near-zero | Auto | ✅ |
| Cloud Bigtable/Firestore | Variable | Low | Auto | ❌ (not MongoDB) |

**Why Atlas:**
- Start on M0 (free) for QA/Staging; promote to M10 for prod with one click.
- Atlas Network Peering with our GCP VPC means MongoDB traffic never leaves Google's backbone.
- Atlas handles backups, point-in-time recovery, and rolling upgrades.
- Connection string stored in GCP Secret Manager; app never sees credentials in config files.

---

## 3. Networking

### VPC Layout

```
VPC: cloudeagle-vpc (10.0.0.0/16)
│
├── Subnet: sync-service-gke     (10.0.1.0/24)  — GKE pods
├── Subnet: sync-service-svc     (10.0.2.0/24)  — GKE services (ClusterIP range)
├── Subnet: mgmt                 (10.0.10.0/24) — Jenkins, bastion
└── VPC Peering ──→ MongoDB Atlas VPC (atlas-peering)
```

### Ingress Path

```
Internet
    │
    ▼
Cloud Armor WAF (rate limiting, geo-blocking, OWASP rules)
    │
    ▼
Global HTTPS Load Balancer (managed TLS via Google-managed cert)
    │
    ▼
GKE Ingress (NGINX ingress controller)
    │
    ▼
sync-service pods (ClusterIP service)
```

- **No public IPs on pods or nodes** — Autopilot nodes are private; only the LB has a public IP.
- **Cloud Armor** sits in front for DDoS protection and rate limiting at zero extra ops.
- **Internal services** (health checks, metrics scraping) use ClusterIP — never exposed externally.

### DNS

Cloud DNS manages `*.cloudeagle.io`. Each environment gets a subdomain:
- `qa.sync.cloudeagle.io` → QA GKE ingress IP
- `staging.sync.cloudeagle.io` → Staging GKE ingress IP
- `sync.cloudeagle.io` → Prod GKE ingress IP

---

## 4. Secrets & IAM

### Secrets Storage: GCP Secret Manager

All secrets (MongoDB URI, API keys, JWT secrets) live in Secret Manager. Zero secrets in code, Kubernetes YAML, or environment variables at rest.

Secrets are mounted into pods as environment variables at pod start using the [GCP Secret Manager CSI driver](https://github.com/GoogleCloudPlatform/secrets-store-csi-driver-provider-gcp):

```yaml
volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: sync-service-secrets
```

### IAM — Principle of Least Privilege

| Identity | Role | Scope |
|----------|------|-------|
| GKE SA (sync-service) | `roles/secretmanager.secretAccessor` | Only `sync-service-*` secrets |
| Jenkins SA | `roles/artifactregistry.writer` | GAR repo only |
| Jenkins SA | `roles/compute.instanceGroupAdmin` | Per-env MIGs only |
| Developer | `roles/container.developer` | Namespace only (not cluster-admin) |
| Prod deploy SA | `roles/compute.loadBalancerAdmin` | Prod backend service only |

Workload Identity binds the Kubernetes service account to the GCP service account — no key files inside the cluster.

---

## 5. Auto-Scaling

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sync-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sync-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
```

- **minReplicas: 2** ensures HA across zones even at low load.
- Scale-up triggered at 60% CPU (conservative to avoid latency spikes during scaling).
- GKE Autopilot provisions new nodes automatically to fit new pods — no manual node sizing.

---

## 6. Logging & Monitoring

### Stack

| Concern | Tool | Rationale |
|---------|------|-----------|
| Application logs | Cloud Logging (GKE auto-ships) | Zero config; structured JSON logs from Spring Boot |
| Metrics | Cloud Monitoring + GKE metrics | CPU, memory, request rates, error rates out of the box |
| APM / Traces | Cloud Trace (Spring Micrometer integration) | Latency breakdown across service calls |
| Uptime checks | Cloud Monitoring uptime checks | Alert if `/actuator/health` fails from 3 regions |
| Alerting | Cloud Monitoring → PagerDuty / Slack | Error rate > 1%, p99 latency > 2s, pod crash loop |
| Dashboards | Cloud Monitoring dashboards | SLI/SLO tracking |

### Key Alerts

```
- Error rate > 1% for 5 min → PagerDuty P1
- p99 latency > 2s for 5 min → Slack #oncall
- Pod restarts > 3 in 10 min → PagerDuty P2
- MongoDB Atlas connection failures → PagerDuty P1
- Secret Manager access denied → Security Slack channel
```

Spring Boot is configured to emit structured JSON logs:
```yaml
logging:
  structured:
    format:
      console: ecs   # Elastic Common Schema — works with Cloud Logging filters
```

---

## 7. Cost Estimate (Startup Tier)

| Resource | Spec | Est. Monthly Cost |
|----------|------|-------------------|
| GKE Autopilot (2–4 pods avg) | 0.5 vCPU / 1 GB per pod | ~$40–80 |
| Cloud Load Balancer | 1 rule + data processing | ~$20 |
| MongoDB Atlas M10 (prod) | 2 vCPU, 2 GB RAM | ~$57 |
| MongoDB Atlas M0 (QA+Staging) | Free tier | $0 |
| Artifact Registry | ~10 GB images | ~$2 |
| Secret Manager | < 10k accesses/mo | ~$1 |
| Cloud Logging / Monitoring | Default free tier | ~$0–10 |
| **Total (prod)** | | **~$120–170/mo** |

This is well within startup constraints, scales linearly with load, and has no idle-node waste.

---

## 8. Architecture Diagram

See `infra/architecture-diagram.svg` (or the rendered version in this repo's README).

The diagram covers:
- Public internet → Cloud Armor → HTTPS LB → GKE Autopilot cluster
- GKE pods → Secret Manager (via Workload Identity)
- GKE pods → MongoDB Atlas (via VPC peering, private connection)
- Jenkins (on mgmt subnet) → Artifact Registry → GKE deploy
- Cloud Logging / Monitoring receiving pod telemetry
