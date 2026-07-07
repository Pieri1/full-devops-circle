# ARCHITECTURE.md – Arquitetura do Sistema

Visão geral da arquitetura do projeto Notification Service com todos os componentes, fluxos e decisões.

---

## Overview Diagrama (High-Level)

```
┌─────────────────────────────────────────────────────────────────┐
│                         GitHub                                   │
│  ┌────────────────────┬─────────────────────────────────────┐   │
│  │ app repo           │ app-config repo (manifestos K8s)    │   │
│  │ (Go source)        │ (Kustomize + Rollouts)             │   │
│  └────────────────────┴─────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
           ↓ (push)              ↓ (pull)
    ┌─────────────────┐    ┌───────────────┐
    │ GitHub Actions  │    │   ArgoCD      │
    │ (CI Pipeline)   │    │   (GitOps)    │
    └────────┬────────┘    └───────┬───────┘
             │                     │
      ┌──────┴─────┐        ┌──────┴──────┐
      ↓            ↓        ↓             ↓
   ┌─────┐    ┌───────┐  ┌──────────────────────┐
   │Lint │    │Build  │  │ Argo Rollouts        │
   │Test │    │Scan   │  │ (Canary Deploy)      │
   └──────┘    │Push  │  │ + Prometheus Analysis│
              │ECR   │  └──────────┬───────────┘
              └────┬─┘             │
                   │               ↓
                  ECR         ┌──────────────────┐
                   │          │  AWS EKS Cluster │
                   └─────────→├─────────┬────────┤
                               │ Prod NS │
                               │ ────────│
                               │ - App   │
                               │ - Loki  │
                               │ - Prom  │
                               └────────┬────────┘
                                     ↓
                    ┌─────────────────────────────┐
                    │  AWS (RDS, SQS, Secrets)    │
                    │  + CloudWatch Monitoring    │
                    └─────────────────────────────┘
```

---

## Componentes Principais

### 1. **Application Layer** (Notification Service)

```
┌─────────────────────────────────────────────────────────────┐
│                 Notification Service (Go)                    │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  HTTP Server (8080)                                           │
│  ├── POST /v1/notifications          [idempotent]            │
│  ├── GET  /v1/notifications/{id}                             │
│  ├── DELETE /v1/notifications/{id}                           │
│  ├── GET /health/live                [k8s liveness]          │
│  └── GET /health/ready               [k8s readiness]         │
│                                                               │
│  Middleware:                                                  │
│  ├── RequestID (trace correlation)                           │
│  ├── OpenTelemetry tracing                                   │
│  ├── Prometheus metrics                                      │
│  └── Request/Response logging (to stdout → Loki)             │
│                                                               │
│  Handler Layer:                                               │
│  ├── NotificationHandler (HTTP contracts)                    │
│  └── NotificationService (business logic)                    │
│                                                               │
│  Repository Layer:                                            │
│  ├── PostgreSQL (persistence)                                │
│  ├── SQS (outbound queue)                                    │
│  └── Secrets Manager (config)                                │
│                                                               │
│  Workers (Goroutines):                                        │
│  ├── SQS Consumer (async delivery)                           │
│  ├── DB Cleanup (old records)                                │
│  └── Health Check ticker                                     │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Key Features:**
- Graceful shutdown (30s grace period).
- Circuit breaker para DB/SQS.
- Retry exponential backoff.
- Connection pooling (20 DB, 10 SQS workers).
- Zero goroutine leaks (tested with goleak).

---

### 2. **Data Layer** (AWS Services)

```
┌────────────────────────────────────────────────────────────┐
│                      AWS Services                           │
├────────────────────────────────────────────────────────────┤
│                                                              │
│ ┌──────────────────┐  ┌────────────────────┐                │
│ │ RDS PostgreSQL   │  │ AWS SQS (Standard) │                │
│ │ ────────────────│  │ ─────────────────  │                │
│ │ • notifications  │  │ • notification-q   │                │
│ │   table          │  │ • DLQ: failed-q    │                │
│ │ • Encryption:    │  │ • Retention: 4 days│                │
│ │   KMS-managed    │  │ • Visibility: 30s  │                │
│ │ • Backup: daily  │  │ • Long polling: 20s│                │
│ │ • Multi-AZ       │  └────────────────────┘                │
│ └──────────────────┘                                         │
│                                                              │
│ ┌──────────────────┐  ┌────────────────────┐                │
│ │ Secrets Manager  │  │ S3 (State + Logs)  │                │
│ │ ─────────────────│  │ ───────────────   │                │
│ │ • db-password    │  │ • TF state         │                │
│ │ • api-key        │  │ • SBOM artifacts   │                │
│ │ • rotation: q90d │  │ • Lifecycle: Glacier│                │
│ └──────────────────┘  └────────────────────┘                │
│                                                              │
└────────────────────────────────────────────────────────────┘
```

**LocalStack (Local Dev):**
- SQS local URI: `http://localstack:4566`
- RDS local URI: `postgresql://...@postgres:5432`
- Secrets local: `.env.local` file

---

### 3. **Kubernetes (EKS / K3s)**

```
┌───────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster (EKS/K3s)               │
├───────────────────────────────────────────────────────────────┤
│                                                                 │
│  Namespace: production                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Notification Service Workloads                           │  │
│  │ ├── Rollout (Argo) – Canary deployment                   │  │
│  │ ├── Service (ClusterIP) – notification-service:8080      │  │
│  │ ├── ConfigMap – app config (no secrets)                  │  │
│  │ ├── Secret (External Secrets) – DB, API keys            │  │
│  │ └── NetworkPolicy – Ingress/Egress rules                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Namespace: monitoring                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ kube-prometheus-stack (Helm)                             │  │
│  │ ├── Prometheus StatefulSet (30-day retention)            │  │
│  │ ├── Grafana Deployment + PVC                             │  │
│  │ ├── Alertmanager StatefulSet                             │  │
│  │ └── PrometheusRule – Alerts + Recording rules            │  │
│  │                                                           │  │
│  │ Loki Deployment                                          │  │
│  │ ├── Loki StatefulSet                                     │  │
│  │ ├── Promtail DaemonSet (sidecar per node)               │  │
│  │ └── PVC (30-day hot, S3 warm)                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Namespace: argocd                                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ ArgoCD Deployment                                        │  │
│  │ ├── ArgoCD API + UI                                      │  │
│  │ ├── ApplicationSet – auto-sync from git                  │  │
│  │ └── Notification Controller (Slack webhook)              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Namespace: litmus (Chaos)                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ LitmusChaos Operator                                     │  │
│  │ ├── ChaosEngine (defines experiment)                     │  │
│  │ └── ChaosExperiment (pod delete, latency, etc)           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Namespaces: default                                            │
│  ├── LocalStack (svc) – SQS, RDS, Secrets                      │
│  └── Docker Registry (svc, optional)                           │
│                                                                 │
│  Node Pool (EKS Prod):                                          │
│  ├── On-demand nodes (2x t3.medium) – App workloads            │
│  └── Spot nodes (2x t3.large) – Non-critical, 60% cheaper      │
│                                                                 │
└───────────────────────────────────────────────────────────────┘
```

**Key Objects:**
- **Rollout (Argo):** Progressive deployment with automated rollback.
- **Service:** Load balancer (internal ClusterIP).
- **ConfigMap:** Non-sensitive app config.
- **Secret (ESO):** Secrets synced from AWS Secrets Manager.
- **NetworkPolicy:** Deny-all inbound except ingress, specific egress.

---

### 4. **CI/CD Pipeline**

```
┌─────────────────────────────────────────────────────────────┐
│            GitHub Actions Workflow (CI)                      │
└─────────────────────────────────────────────────────────────┘

Trigger: push to PR / merge to main

Parallel Jobs:
├─ Lint (1 min)
│  ├── golangci-lint
│  ├── terraform validate
│  └── yamllint
│
├─ SAST (2 min)
│  ├── semgrep
│  └── trivy config scan
│
├─ Secrets (1 min)
│  └── gitleaks
│
├─ Tests (3 min)
│  ├── go test -race (unit tests)
│  └── testcontainers (integration)
│
└─ Build (5 min)
   ├── docker buildx (multi-platform)
   ├── trivy image scan
   ├── syft SBOM generation
   ├── cosign image signing
   └── push to ECR

Post-CI (if main branch):
└─ Infracost (if terraform changes)
   └── comment on PR with delta cost

CI Artifacts:
├── Docker image in ECR (tagged: latest, v1.2.3, git-abc1234)
├── SBOM (sbom.json)
└── Image signature (Cosign)
```

**Timing:**
- Lint + SAST + Secrets: ~4 min (parallel).
- Tests: ~3 min.
- Build + Scan: ~5 min.
- **Total CI time: ~10 min** (not critical path).

---

### 5. **GitOps Deployment Flow**

```
┌──────────────────────────────────────────────────────────────┐
│  Deployment Sequence (GitOps)                                 │
└──────────────────────────────────────────────────────────────┘

T0 - Merge PR to main (app repo)
  └→ GitHub Actions triggers CI
    └→ Tests pass, image pushed to ECR with tag v1.2.3

T5 - CI creates PR in app-config repo
  └→ Updates overlays/prod/kustomization.yaml:
    ```yaml
    images:
    - name: notification-service
      newTag: v1.2.3  # ← Updated
    ```

T6 - PR in app-config is auto-merged (by bot)
  └→ Commit SHA: a1b2c3d4

T7 - ArgoCD detects commit
  └→ Syncs Application (pull model)
    └→ Deploys Argo Rollout with new image

T8-T23 - Argo Rollouts Canary Analysis
  ├─ T8-T13: 10% traffic to new pods (6 min)
  │         Prometheus analyzes metrics every 60s
  │         If error% > 1% OR p99 latency > 500ms → abort
  │
  ├─ T13-T18: 50% traffic
  │          Analysis continues
  │
  └─ T18-T23: 100% traffic
              Analysis continues for final 5 min

Result:
├─ If all probes pass → Fully deployed ✓
├─ If metrics violated → Automatic rollback to v1.2.2 ✗
└─ Slack notification either way

Observability:
├─ Grafana dashboard shows deployment progress
├─ Loki shows logs from old/new pods
├─ Jaeger shows trace correlation (trace_id)
└─ Prometheus alerts if canary aborted (PagerDuty page)
```

---

### 6. **Observability Stack**

```
┌────────────────────────────────────────────────────────┐
│           Observability (Three Pillars)                 │
├────────────────────────────────────────────────────────┤
│                                                          │
│ METRICS (Prometheus)                                    │
│ ┌────────────────────────────────────────────────────┐  │
│ │ • http_requests_total                              │  │
│ │ • http_request_duration_seconds (histogram)        │  │
│ │ • db_connections_active (gauge)                    │  │
│ │ • sqs_messages_received_total                      │  │
│ │ • notifications_sent_total (business metric)       │  │
│ │ • chaos_experiment_running (flag)                  │  │
│ │                                                    │  │
│ │ Scrape targets:                                    │  │
│ │ • notification-service:8080/metrics (15s)         │  │
│ │ • prometheus-operator (K8s metrics)               │  │
│ │ • kubelet (node metrics)                          │  │
│ │ • kube-proxy (cluster metrics)                    │  │
│ └────────────────────────────────────────────────────┘  │
│                                                          │
│ LOGS (Loki)                                              │
│ ┌────────────────────────────────────────────────────┐  │
│ │ • Promtail collects stdout/stderr from pods       │  │
│ │ • Labels: {namespace="production", pod=..., ...}  │  │
│ │ • Hot: 30 days (SSD, fast query)                  │  │
│ │ • Warm: S3 (cheaper, slower)                      │  │
│ │ • Query: level="error" AND msg~"database"        │  │
│ │ • Alerting: LogQL queries → alert rules          │  │
│ └────────────────────────────────────────────────────┘  │
│                                                          │
│ TRACES (Jaeger / X-Ray)                                  │
│ ┌────────────────────────────────────────────────────┐  │
│ │ • Instrumentation: OpenTelemetry Go SDK           │  │
│ │ • Spans: HTTP handler, DB queries, SQS ops       │  │
│ │ • Sampling: 10% (cost optimization)              │  │
│ │ • W3C Trace Context propagation                  │  │
│ │ • Exporter: Jaeger (dev), X-Ray (prod)          │  │
│ │ • Retention: 7 days                              │  │
│ │ • Service map: Auto-generated                    │  │
│ └────────────────────────────────────────────────────┘  │
│                                                          │
│ DASHBOARDS (Grafana)                                     │
│ ├─ Golden Signals (latency, traffic, errors, saturation)│
│ ├─ App Metrics (notifications sent, queue depth)        │
│ ├─ DORA Metrics (lead time, deployment freq)           │
│ ├─ SLO Dashboard (burn rate, budget remaining)         │
│ └─ Infrastructure (CPU, memory, network)               │
│                                                          │
│ ALERTING                                                 │
│ ├─ Prometheus AlertManager routes alerts               │
│ ├─ Slack #alerts channel                               │
│ ├─ PagerDuty on-call (critical only)                   │
│ └─ Email (CFO budget alerts)                           │
│                                                          │
└────────────────────────────────────────────────────────┘
```

---

## Data Flow: Request Lifecycle

```
User/Client
  │
  ├─ HTTP POST /v1/notifications
  │  └─ Payload: {email, title, body, idempotency_key}
  │
  ├─ [OpenTelemetry] Span Start (trace_id, span_id)
  │
  ├─ [Middleware] RequestID + Logging
  │  └─ Log: "Request received: {trace_id, correlation_id}"
  │
  ├─ [Handler] Validation
  │  └─ Check: idempotency_key, required fields
  │
  ├─ [Service] Business Logic
  │  └─ Check: idempotency (query DB for key)
  │     └─ If exists: return cached result (idempotent)
  │     └─ If new: create record in PostgreSQL
  │
  ├─ [Repository] DB Write
  │  ├─ Query: INSERT INTO notifications (...)
  │  ├─ Log: "Notification created: {id}"
  │  └─ Span: "db.query" (query, duration)
  │
  ├─ [Service] Queue to SQS
  │  ├─ Send message to SQS queue
  │  ├─ Payload: {notification_id, retry_count}
  │  ├── Log: "Queued for delivery"
  │  └─ Span: "sqs.send_message"
  │
  ├─ [Handler] Response
  │  └─ HTTP 202 Accepted: {id, status: "pending"}
  │
  ├─ [Middleware] Prometheus Metrics
  │  ├─ http_requests_total{method="POST", status="202"}++
  │  └─ http_request_duration_seconds{...} observe(1.5s)
  │
  ├─ [OpenTelemetry] Span End
  │  └─ Send span to Jaeger/X-Ray
  │
  └─ Response back to client

---

Async: SQS Consumer (separate goroutine)
  │
  ├─ Long-poll SQS (20s wait)
  │  └─ Receive message
  │
  ├─ [OpenTelemetry] Extract trace_id from message metadata
  │  └─ Span context: Continuation of original request trace
  │
  ├─ [Service] Send email (3rd party API)
  │  ├─ Retry logic: exponential backoff (1s, 2s, 4s, 8s)
  │  ├─ CircuitBreaker: 5 failures → open (30s)
  │  └─ Timeout: 10s
  │
  ├─ [Repository] Update notification status
  │  ├─ Query: UPDATE notifications SET status = 'delivered'
  │  └─ Timestamp: delivered_at
  │
  ├─ [SQS] Delete message from queue
  │  └─ Message ack
  │
  ├─ [Metrics]
  │  └─ notifications_sent_total{status="delivered"}++
  │
  ├─ [Logging]
  │  └─ Log: "Email delivered: {id}"
  │
  └─ [DLQ] If max retries exceeded
     └─ Move to DLQ for manual review
```

---

## Security Layers

```
┌─────────────────────────────────────────────────────┐
│         Security Model (Defense in Depth)           │
├─────────────────────────────────────────────────────┤
│                                                      │
│ Layer 1: Source Code (Shift-Left)                   │
│ ├─ Pre-commit: Gitleaks (block secrets)            │
│ ├─ CI: Semgrep (SAST), Trivy (SCA)                │
│ └─ Code review (GitHub CODEOWNERS)                 │
│                                                      │
│ Layer 2: Build & Registry                           │
│ ├─ Docker multi-stage (minimize image)             │
│ ├─ Non-root user (distroless)                      │
│ ├─ Image signing (Cosign + OIDC)                   │
│ └─ Trivy scan (admission gate)                     │
│                                                      │
│ Layer 3: Kubernetes Admission                       │
│ ├─ OPA/Gatekeeper policies:                        │
│ │  ├─ Non-root pods                                │
│ │  ├─ Resource limits required                     │
│ │  ├─ Signed images only                           │
│ │  └─ No privileged containers                     │
│ │                                                  │
│ └─ Pod Security Policy (deprecated, use PSS)       │
│                                                      │
│ Layer 4: Network                                     │
│ ├─ NetworkPolicy:                                  │
│ │  ├─ Deny all by default                         │
│ │  ├─ Allow ingress from LB only                   │
│ │  └─ Allow egress: DB (5432), SQS API (443)      │
│ │                                                  │
│ └─ Falco runtime detection (anomalies)             │
│                                                      │
│ Layer 5: Runtime (App Code)                         │
│ ├─ Circuit breaker (graceful degradation)         │
│ ├─ Rate limiting (per IP, per user)               │
│ ├─ Input validation (parameterized queries)        │
│ ├─ TLS 1.3 (in-flight)                            │
│ └─ Logging (audit trail)                          │
│                                                      │
│ Layer 6: Data (At-Rest)                             │
│ ├─ Database encryption (KMS-managed keys)         │
│ ├─ Secrets Manager (AES-256)                      │
│ ├─ S3 bucket encryption                           │
│ └─ Backup encrypted                               │
│                                                      │
│ Layer 7: Audit & Compliance                         │
│ ├─ CloudTrail logs all AWS API calls              │
│ ├─ K8s audit logs (create, delete, patch)         │
│ ├─ Gitleaks + TruffleHog in CI                    │
│ └─ Regular penetration testing (q6m)              │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## Resilience Patterns

```
┌──────────────────────────────────────────────────────┐
│        Resilience (Handling Failures)                │
├──────────────────────────────────────────────────────┤
│                                                       │
│ 1. Circuit Breaker                                   │
│    ┌──────────────┐                                  │
│    │ Normal State │ ← 5 failures in 10s → Open      │
│    │ (pass thru)  │                                  │
│    └──────────────┘                                  │
│         ↑                                             │
│         │ ← 30s timeout → Half-Open               │
│         │                                             │
│    ┌──────────────┐                                  │
│    │ Open State   │ ← 1 success → Closed           │
│    │ (reject fast)│   (1 failure → Open)           │
│    └──────────────┘                                  │
│                                                       │
│ 2. Retry Strategy (Exponential Backoff)              │
│    Attempt 1: Fail → Wait 1s                         │
│    Attempt 2: Fail → Wait 2s                         │
│    Attempt 3: Fail → Wait 4s                         │
│    Attempt 4: Fail → Wait 8s                         │
│    Attempt 5: Fail → DLQ (give up)                   │
│                                                       │
│ 3. Timeout                                            │
│    HTTP: 30s (client-level)                          │
│    DB: 5s                                             │
│    SQS: 10s                                           │
│    └─ If timeout → circuit breaker opens            │
│                                                       │
│ 4. Graceful Degradation                              │
│    ├─ DB down → Queue only (no delivery)            │
│    ├─ SQS down → Retry w/ exponential backoff       │
│    └─ Both down → 503 Service Unavailable           │
│                                                       │
│ 5. Graceful Shutdown                                 │
│    SIGTERM received                                  │
│      ├─ Stop accepting new requests (readiness=fail)│
│      ├─ Wait 30s for in-flight requests to finish   │
│      ├─ Drain SQS consumer goroutine                │
│      └─ Close DB connections, exit cleanly         │
│                                                       │
│ 6. Pod Restart Policies                              │
│    ├─ Liveness probe fails → Kill pod               │
│    ├─ Readiness probe fails → Remove from LB        │
│    └─ BackoffLimit=3 → Pod stays failed if restarting│
│                                                       │
│ 7. Horizontal Pod Autoscaling (HPA)                  │
│    Trigger: CPU > 80% OR RPS > 1000/s               │
│      ├─ Scale up: +1 pod (min 2, max 10)           │
│      ├─ Scale down: -1 pod (if idle > 5min)        │
│      └─ Canary unaffected (separate target)        │
│                                                       │
└──────────────────────────────────────────────────────┘
```

---

## Development Workflow (Local)

```
Developer Machine (Linux/macOS/Windows WSL2)
  │
  ├─ Clone repo
  │  └─ Run: `task up`
  │     ├─ Start Dev Container (VS Code)
  │     ├─ Spin up LocalStack (SQS, RDS, Secrets)
  │     ├─ Spin up K3s cluster
  │     ├─ Deploy monitoring stack (Prometheus, Grafana, Loki)
  │     └─ ~10 minutes total
  │
  ├─ Modify code (app/cmd/main.go)
  │  └─ Auto-format (gofmt on save)
  │
  ├─ Run tests locally
  │  ├─ UT: `go test ./...` (uses testify, parallel)
  │  ├─ Integration: Testcontainers-go (postgres container)
  │  └─ Results: Pass/fail + coverage
  │
  ├─ Lint check
  │  └─ `task lint` (golangci-lint, terraform validate, yamllint)
  │
  ├─ Secrets check (pre-commit hook)
  │  └─ Gitleaks (blocks if secret detected)
  │
  ├─ Build & Run Locally
  │  ├─ `task build` (Docker build, creates image)
  │  ├─ Deploy to K3s (manual kustomize build + apply)
  │  └─ Logs: `kubectl logs -f deployment/notification-service`
  │
  ├─ Debug
  │  ├─ F5 in VS Code (Delve debugger)
  │  └─ Connect to pod: `kubectl debug -it pod/...`
  │
  ├─ Test in Staging (optional)
  │  └─ `task deploy-staging` (Push to ECR, deploy to AWS EKS)
  │
  ├─ Commit & Push
  │  ├─ Gitleaks runs (pre-commit)
  │  └─ If clean, push to GitHub
  │
  └─ GitHub Actions CI Runs
     ├─ Lint + Tests + Scan
     ├─ Build image (ECR push)
     └─ Creates PR in app-config repo (auto-merge)
```

---

## Chaos Engineering Scenarios

```
┌────────────────────────────────────────────────────┐
│      Experiment: Pod Delete Under Load             │
├────────────────────────────────────────────────────┤
│                                                     │
│ Setup:                                              │
│ ├─ 2 replicas of notification-service              │
│ ├─ K6 load test: 100 RPS constant                 │
│ ├─ LitmusChaos experiment: kill 1 pod per 30s     │
│ └─ Duration: 60s                                   │
│                                                     │
│ Expected Behavior:                                  │
│ ├─ Pod killed → Immediately evicted from LB       │
│ ├─ New pod scheduled + starts (readiness: 5s)     │
│ ├─ RPS continues: no spike, no error increase     │
│ ├─ Prometheus: latency p99 continues stable       │
│ └─ Result: Zero visible downtime to client       │
│                                                     │
│ Metrics Validation:                                 │
│ ├─ RPS: 100 ± 5 (never dips)                      │
│ ├─ Error rate: 0% (no 5xx)                        │
│ ├─ p99 latency: < 200ms (SLO)                     │
│ └─ Pod restart time: < 10s                        │
│                                                     │
│ Pass Criteria:                                      │
│ ├─ ✓ RPS maintained                               │
│ ├─ ✓ No errors                                    │
│ ├─ ✓ Within SLO                                   │
│ └─ ✓ Automated recovery (no manual intervention)  │
│                                                     │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  Experiment: Canary Deploy + Chaos                 │
├────────────────────────────────────────────────────┤
│                                                     │
│ Scenario:                                           │
│ ├─ Version v1.2.2 running (stable)                │
│ ├─ Canary deploy v1.2.3 starts (10% traffic)      │
│ ├─ Pod delete chaos fires (kill old pod)          │
│ ├─ Canary should continue (resilient)             │
│ └─ Metrics should not spike (new version OK)      │
│                                                     │
│ Timeline:                                           │
│ T0: Canary starts (10% to v1.2.3)                 │
│ T5: Chaos kills v1.2.2 pod                        │
│ T10: New v1.2.2 pod recovered                     │
│ T15: Canary progresses to 50%                     │
│ T25: Canary completes 100%                        │
│ T30: Analysis ends (no violations)                │
│                                                     │
│ Expected:                                           │
│ ├─ Canary NOT aborted (chaos + chaos-during-deploy) │
│ ├─ New version v1.2.3 rolled out fully            │
│ ├─ No elevated error rate                         │
│ └─ Rollback would be automatic if metrics bad     │
│                                                     │
└────────────────────────────────────────────────────┘
```

---

## Summary

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **App** | Go 1.23 | Microservice (API + async workers) |
| **Data** | PostgreSQL + SQS | Persistence + queueing |
| **Container** | Docker (distroless) | Packaging |
| **Orchestration** | Kubernetes (EKS/K3s) | Deployment, scaling, self-healing |
| **Config** | Kustomize | K8s manifests management |
| **GitOps** | ArgoCD | Sync git → cluster (pull model) |
| **Progressive** | Argo Rollouts | Canary deploy + automated rollback |
| **Observability** | Prometheus + Grafana | Metrics & dashboards |
| **Logs** | Loki + Promtail | Centralized logging |
| **Traces** | Jaeger / X-Ray | Distributed tracing |
| **Alerts** | AlertManager | Routing + PagerDuty |
| **Chaos** | LitmusChaos | Failure injection + validation |
| **Security** | OPA/Gatekeeper | Policy-as-code admission |
| **CI/CD** | GitHub Actions | Automated testing & deployment |
| **IaC** | Terraform | Infrastructure provisioning |
| **Secrets** | AWS Secrets Manager | Secret management + rotation |
| **FinOps** | Infracost + Kubecost | Cost tracking |

