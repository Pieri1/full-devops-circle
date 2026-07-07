# Checklist Avançado: Projeto DevOps Full-Cycle

Este documento define as etapas, ferramentas e critérios de aceitação para o desenvolvimento de um ambiente de microserviços de alto padrão, englobando DevEx, Segurança, IaC, GitOps, Observabilidade e Resiliência.

---

## Fase 1: Developer Experience (DevEx) e Fundação
**Objetivo:** Setup < 10min. Qualquer dev (Linux/macOS/Windows) clona, roda `task up` e inicia.
**Plataformas Suportadas:** Linux (Ubuntu 22.04+), macOS (13+), Windows (WSL2).

- [ ] **1.1 Dev Container:** Configurar `.devcontainer/devcontainer.json` com:
  - Imagem base: `mcr.microsoft.com/devcontainers/go:1.23`
  - Features: Docker-in-Docker, Docker Compose, Terraform CLI, Go 1.23, Task CLI, git-flow.
  - Extensions VS Code: Go, Docker, GitLens, REST Client, Kubernetes.
  - VS Code settings: gofmt, goimports, golangci-lint automático.
  - **Critério:** `task up` deve ter cluster K3s rodando em < 2min.

- [ ] **1.2 Taskfile.yml:** Criar tasks para:
  - `task up` – Inicia Dev Container, LocalStack, K3s.
  - `task down` – Para tudo, limpa volumes.
  - `task lint` – golangci-lint, terraform validate, yamllint.
  - `task test` – Go tests com coverage (>80%).
  - `task build` – Build local da imagem Docker.
  - `task docs` – Gera OpenAPI e README.md com diagramas.
  - **Critério:** Nenhuma dependência de PATH global exceto Task CLI.

- [ ] **1.3 LocalStack:** `docker-compose.yml` com:
  - Services: sqs, rds (postgres), s3, secrets-manager, dynamodb.
  - Health checks em cada serviço.
  - Seed data de teste (ex: bancos de dados, queues).
  - **Critério:** Startup < 30s, mesma config que produção.

- [ ] **1.4 Documentação Viva:**
  - `README.md` com Quick Start (3 passos), troubleshooting.
  - `docs/ARCHITECTURE.md` com diagramas Mermaid (componentes, fluxo, deployment).
  - `docs/API.md` com Swagger integrado (ou link para Swagger UI).
  - **Critério:** README atualizado toda mudança de arquitetura.

- [ ] **1.5 VS Code Workspace:**
  - `.vscode/settings.json`: gofmt, goimports, golangci-lint on save.
  - `.vscode/extensions.json`: Go, Docker, GitLens, REST Client, Kubernetes, Dev Containers.
  - `.vscode/launch.json`: Debug config para app local + localhost:5000.
  - **Critério:** F5 (debug) deve conectar no app rodando em K3s em <5s.

---

## Fase 2: Aplicação e Shift-Left Security (DevSecOps)
**Objetivo:** API resiliente, performática (< 100ms p99), totalmente testável e segura desde o primeiro commit.
**Linguagem:** Go 1.23 (decisão vinculante).

- [ ] **2.1 Estrutura Go:** Usar standard layout:
  - `cmd/` – Main application (HTTP server + graceful shutdown).
  - `internal/` – Business logic (handler, service, repository).
  - `pkg/` – Reusable libraries (logging, config, telemetry).
  - `migrations/` – Database schemas (Flyway via Go embed).
  - **Critério:** Build time < 10s, binary size < 50MB (distroless).

- [ ] **2.2 Microserviço Base – Notificações:**
  - **Endpoints:**
    - `POST /v1/notifications` – Send notification (idempotent, x-idempotency-key).
    - `GET /v1/notifications/{id}` – Get status.
    - `DELETE /v1/notifications/{id}` – Cancel pending.
    - `GET /health/live` – Liveness probe.
    - `GET /health/ready` – Readiness probe (checks DB, SQS, secrets).
  - **Middleware:** Request/response logging, correlation ID, timeout 30s.
  - **Database:** PostgreSQL (RDS). Schema migrations on startup.
  - **Queue:** AWS SQS para async delivery. Consumer em goroutine separada.
  - **Secrets:** AWS Secrets Manager via SDK. Rotate em cache q12h.
  - **Critério:** 
    - Health checks respondem em < 100ms.
    - Idempotência garantida (banco de dados).
    - Zero goroutine leaks (test com goleak).

- [ ] **2.3 Versionamento de API:**
  - Usar `/v1/` no path (não Accept header). Facilita cache, debugging, logs.
  - **Stratégia de deprecação:** Maintém v1 + v2 em paralelo por 6 meses. Warning header em responses.
  - **Critério:** Client nunca quebra sem aviso prévio (SLA de 2 versões).

- [ ] **2.4 Testes Completos:**
  - **UT:** `testify/assert`, coverage > 80%, paralelos (`t.Parallel()`).
  - **Integration:** `testcontainers-go` (PostgreSQL + SQS local).
    - Docker deve estar disponível (já existe em Dev Container).
    - Tests rodam com `docker run -v $(pwd):/work ...`.
  - **Contract Testing:** Pact + broker para futuros serviços (setup mas não usado ainda).
  - **Performance:** Benchmark key operations (`BenchmarkNotificationHandler` com `go test -bench`).
  - **Critério:** `task test` = UT + integration, tudo passa em < 2min.

- [ ] **2.5 Proteção de Segredos:**
  - **Gitleaks** em `.pre-commit-config.yaml`:
    - Bloqueia patterns: AWS KEY, senha, private key, token.
    - Skip patterns para dados de teste (ex: `AKIA...REDACTED`).
  - **GitHub Actions:** Gitleaks roda no CI também (double-check).
  - **Ambiente:** Usar `.env.example` (no git). Secrets no `.env.local` (gitignored).
  - **Critério:** Nenhum secret vaza em git. CI rejeita PR se detected.

- [ ] **2.6 Análise Estática (SAST):**
  - **Semgrep:** Regras para Go (owasp-go, go-best-practices).
    - Configurar `.semgrep.yml` com paths exclusos (vendor, test mocks).
  - **golangci-lint:** `staticcheck`, `gosec`, `errcheck`, `ineffassign`.
    - `.golangci.yml` com presets strict.
  - **Local:** `task lint` roda ambos. Falha se encontra issues.
  - **CI:** GitHub Actions roda antes de merge.
  - **Critério:** Sem warnings em main branch.

- [ ] **2.7 Containerização Otimizada:**
  - **Dockerfile multi-stage:**
    ```dockerfile
    # Builder
    FROM golang:1.23-alpine AS builder
    COPY . /app
    RUN cd /app && go build -ldflags="-s -w" -o app ./cmd/notification-service
    
    # Runtime
    FROM gcr.io/distroless/base-debian12
    COPY --from=builder /app/app /app
    USER nonroot:nonroot
    EXPOSE 8080
    CMD ["/app"]
    ```
  - **Imagem:** < 50MB, sem shell, sem package manager, non-root.
  - **Build:** Docker Buildx multi-platform (linux/amd64, linux/arm64).
  - **Critério:** Image scan (Trivy) sem vulnerabilidades críticas.

- [ ] **2.8 OpenTelemetry (Instrumentação):**
  - **Lib:** `go.opentelemetry.io` SDK.
  - **Spans:** HTTP middleware (request duration), Database queries, SQS operations.
  - **Exporter:** OTEL Collector via Docker (dev), AWS X-Ray (prod).
  - **Propagation:** W3C Trace Context header propagado em SQS messages.
  - **Critério:** Todas requisições têm `trace_id` em logs e spans (correlação perfeita).

---

## Fase 3: Infraestrutura como Código (IaC) e FinOps
**Objetivo:** 100% IaC, modular, testável, custo rastreável e previsível.
**Provisionador:** Terraform 1.6+.

- [ ] **3.1 Estrutura de Módulos Terraform:**
  - `infra/terraform/modules/vpc/` – VPC, subnets (pub+priv), NAT Gateway, IGW, route tables.
  - `infra/terraform/modules/eks/` – EKS cluster, node groups (spot + on-demand), IAM roles.
  - `infra/terraform/modules/rds/` – RDS PostgreSQL, security group, encryption, backups (retention 30d).
  - `infra/terraform/modules/secrets/` – Secrets Manager, key rotation q90d.
  - `infra/terraform/modules/monitoring/` – IAM roles para Prometheus scrape, CloudWatch.
  - `infra/terraform/modules/networking/` – Load Balancer, security groups, Network ACLs.
  - **Variables & Outputs:** Cada módulo tem `variables.tf` (inputs), `outputs.tf` (consumption), `terraform.tfvars.example`.
  - **Critério:** Modular o suficiente para reusar em múltiplos projetos (ex: networking é genérico).

- [ ] **3.2 Ambientes (environments):**
  - `infra/terraform/environments/local/` – LocalStack (dev). Tagging: `environment=local`, `cost-center=r&d`.
  - `infra/terraform/environments/dev/` – AWS dev (small instances). Tagging: `environment=dev`, `cost-center=dev`.
  - `infra/terraform/environments/prod/` – AWS prod (HA). Tagging: `environment=prod`, `cost-center=prod`.
  - Cada env tem `main.tf`, `variables.auto.tfvars` (sobrescreve locals).
  - **Critério:** Mesmo código-base, diferentes inputs por env (DRY).

- [ ] **3.3 Validação de IaC (Shift-Left):**
  - **tfsec:** Escaneia Terraform por inseguridades.
    - `.tfsec/config.json` com skip de falsos positivos (ex: dev env sem encryption).
  - **Trivy:** Scan de configs Kubernetes + Terraform.
  - **Terraform Validate:** `terraform validate` em CI.
  - **Local:** `task lint` roda tfsec + terraform validate.
  - **Critério:** Sem findings críticos. Warnings investigados antes de merge.

- [ ] **3.4 FinOps (Custo Controlado):**
  - **Infracost CLI:**
    - `.infracost.yml` global com pricing rules.
    - GitHub Actions roda em cada PR: `infracost comment --pull-request $PR_ID`.
    - Exemplo: "Terraform changes: +$150/month (RDS t4g.medium + 2 nodes t3.medium)".
  - **Kubecost (prod):** Dashboard em Grafana mostrando custo por namespace/pod/app.
  - **Guardrails:** Rejeitar PRs com delta > $500/month (configurável).
  - **Budget Alerts:** SNS alert se mês acumular > budget (ex: $2k).
  - **Critério:** Todo dev sabe o custo de suas mudanças antes de merge.

- [ ] **3.5 Estado Remoto (Terraform State):**
  - **Backend S3 + DynamoDB:**
    - `terraform {backend "s3" {...}}` com bucket versioned, encrypted, private.
    - DynamoDB table para state locking (evita race conditions).
    - Acesso via IAM role (GitHub Actions assume role via OIDC).
  - **LocalStack (dev):** S3 + DynamoDB simulados. Mesma estrutura que prod.
  - **State Locking:** `terraform apply` aguarda lock. Timeout 5min.
  - **Backup:** State snapshots daily em S3 Glacier (30 dias retention).
  - **Critério:** State nunca commitado em git. Nunca manualmente editado.

- [ ] **3.6 Database Migrations:**
  - **Tool:** Flyway (SQL-based, agnóstico).
  - `db/migrations/` com versionamento: `V1__init.sql`, `V2__add_column.sql`.
  - Rodado via init container em K8s (antes de app startup).
  - **Validação:** Undo scripts (`U2__...sql`) para rollback.
  - **Critério:** Migrations testadas localmente antes de deploy.

- [ ] **3.7 Secrets Management (Production):**
  - **AWS Secrets Manager:** Store chaves de DB, API keys, certs SSL.
  - **External Secrets Operator (K8s):** Sync secrets do AWS para K8s Secrets.
  - **Rotation:** Automática q90d (AWS Secrets Manager rotator Lambda).
  - **Audit:** CloudTrail logs all secret access.
  - **Local (dev):** Secrets em `.env.local` (gitignored). Testcontainers injeta.
  - **Critério:** Nenhum secret em git. Rotation sem downtime (Zero-Downtime Secrets).

---

## Fase 4: Integração e Entrega Contínuas (CI/CD & GitOps)
**Objetivo:** Toda mudança em main é deployada automaticamente em canário 10min depois. Rollback automático se métrica > SLO.
**Plataforma:** GitHub Actions. Repositórios: app (este) + config (manifestos K8s separados).

- [ ] **4.1 Pipeline de CI (GitHub Actions):**
  - **Trigger:** Push em PR, merge em main.
  - **Jobs (paralelos):**
    - **Lint:** `golangci-lint`, `terraform validate`, `yamllint`, `shellcheck`.
    - **SAST:** `semgrep`, `trivy config`.
    - **Secrets:** `gitleaks`.
    - **Tests:** UT (go test -race), Integration (testcontainers).
    - **Coverage:** Generate report, fail if < 80%.
  - **Build & Push:**
    - Build multi-platform image (linux/amd64 + arm64) com Docker Buildx.
    - Tag: `latest`, `v${VERSION}`, `git-${SHA:0:7}`.
    - Push para ECR (AWS).
  - **Security Scans:**
    - **Trivy:** Image scan → Sarif report.
    - **SBOM:** Syft generate SBOM, upload para artifact.
    - **Signing:** Cosign sign image com OIDC token GitHub.
  - **FinOps:**
    - Se PR com Terraform changes: Infracost comment no PR com delta.
  - **Critério:**
    - CI duration < 10min.
    - Fail fast: Lint antes de tests, build antes de scan.
    - Sem manual approval necessário (somente automático).

- [ ] **4.2 Repositório de Manifestos (GitOps):**
  - **Repo separado:** `app-config` (GitHub).
  - **Estrutura:**
    ```
    app-config/
    ├── base/
    │   ├── kustomization.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── configmap.yaml
    │   └── ...
    ├── overlays/
    │   ├── dev/
    │   ├── staging/
    │   └── prod/
    └── .sync/
        └── image-update-log.txt
    ```
  - **Kustomize** para env-specific overrides (replicas, resources, replicas).
  - **Image Update:** CI (app repo) abre PR em app-config com nova tag:
    ```yaml
    # base/deployment.yaml
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/notification-service:v1.2.3
    ```
  - **Critério:** Image tag sempre atualizado. ArgoCD sincroniza automaticamente (pull model).

- [ ] **4.3 ArgoCD Setup & Sync:**
  - **Install:** Via Helm em K8s (prod/staging). Chart: `argo-cd/argo-cd`.
  - **Repositories:** Conectar `app-config` repo com deploy key (read-only).
  - **Applications:** 
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: notification-service
      namespace: argocd
    spec:
      project: production
      source:
        repoURL: https://github.com/org/app-config
        targetRevision: HEAD
        path: overlays/prod
      destination:
        server: https://kubernetes.default.svc
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
    ```
  - **Sync Mode:** Automático (self-healing). Se manifesto no K8s divergir de repo → sincroniza.
  - **Notification:** ArgoCD notifica Slack quando sync sucede/falha.
  - **Critério:** Deploy é pull-based. Nenhum `kubectl apply` manual. ArgoCD é fonte da verdade.

- [ ] **4.4 Progressive Delivery (Canary):**
  - **Argo Rollouts:** Substituir `Deployment` por `Rollout` no app-config.
  - **Strategy Canary:**
    ```yaml
    spec:
      strategy:
        canary:
          steps:
          - setWeight: 10
          - pause: {duration: 5m}
          - setWeight: 50
          - pause: {duration: 5m}
          - setWeight: 100
      canaryService: notification-service-canary
      stableService: notification-service-stable
    ```
  - **Análise Automática:** Rollouts monitora prometheus durante canary.
    ```yaml
    analysis:
      - name: success-rate
        interval: 60s
        threshold: 95
        query: rate(requests_total{job="notification"}[5m])
    ```
  - **Rollback:** Se p99 latência > 500ms OU error rate > 1% → abort canary automaticamente.
  - **Critério:** Rollout duration max 15min. Usuários no 10% não veem erro (ou < 0.1%).

- [ ] **4.5 Container Registry (ECR):**
  - **AWS ECR:** Private registry em `123456789.dkr.ecr.us-east-1.amazonaws.com`.
  - **Image Lifecycle:** Delete untagged images após 7 dias. Mantém últimas 10 tags.
  - **Scanning:** Trivy scan automático on push. Fail if critical vulnerability.
  - **Access:** GitHub Actions assume IAM role via OIDC para push.
  - **Critério:** Nenhuma imagem pública. Assinadas com Cosign. Vulnerabilidades rastreadas.

- [ ] **4.6 Policy as Code (OPA/Gatekeeper):**
  - **Validação em tempo de sync (ArgoCD):**
    ```yaml
    # rego rules
    deny[msg] {
        input.request.object.spec.containers[_].securityContext.runAsNonRoot == false
        msg := "Containers must run as non-root"
    }
    ```
  - **Regras obrigatórias:**
    - Non-root containers.
    - Resource limits (CPU/Memory).
    - Probe (liveness + readiness).
    - Image pull policy = Always.
  - **Falha:** Rejeta manifesto se policy viola. Msg clara no PR.
  - **Critério:** Zero policy violations em prod.

---

## Fase 5: Observabilidade Completa (O11y)
**Objetivo:** Responder em < 30s: "Está lento?", "Está falhando?", "Onde exatamente?". SLO = 99.95% uptime, p99 < 200ms.
**Stack:** Prometheus + Grafana + Loki + Jaeger (dev), X-Ray (prod).

- [ ] **5.1 Prometheus + Alertmanager:**
  - **Deploy:** Via Helm chart `kube-prometheus-stack` em namespace `monitoring`.
  - **Scrape Config:**
    - `notification-service:8080/metrics` – app metrics (custom + golang defaults).
    - `kube-proxy`, `kubelet` – node/cluster metrics.
    - Interval: 15s (tradeoff: resolution vs storage).
  - **Storage:** Prometheus retention = 30 dias (SSD, 50GB inicial).
  - **Recording Rules:** Pre-compute expensive queries:
    ```yaml
    - alert: HighLatency
      expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 0.2
      for: 1m
      annotations:
        summary: "p99 latency > 200ms"
    ```
  - **AlertManager:** Routes alerts to Slack #alerts, PagerDuty (critical).
  - **Critério:** Prometheus queries < 500ms even with 30d data.

- [ ] **5.2 Grafana Dashboards:**
  - **Golden Signals:**
    - **Latency:** p50, p95, p99 (histograma). Query: `histogram_quantile(X, rate(http_request_duration_seconds_bucket[5m]))`.
    - **Traffic:** RPS (requests/sec). Query: `rate(http_requests_total[1m])`.
    - **Errors:** Error rate %. Query: `100 * rate(http_requests_total{status=~"5.."}[1m]) / rate(http_requests_total[1m])`.
    - **Saturation:** CPU/Memory %. Query: `100 * node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes`.
  - **Application Metrics:**
    - DB connection pool usage.
    - SQS queue depth.
    - Cache hit ratio (redis).
  - **DORA Metrics:**
    - **Lead Time:** Commit → Deploy. Query GitHub API via webhook (custom metric).
    - **Deployment Frequency:** Deploys per day/week. Query ArgoCD API.
    - **MTTR (Mean Time to Recover):** Tempo entre alert → resolved. From AlertManager history.
    - **Change Failure Rate:** Failed deployments / total. Query ArgoCD.
  - **Critério:** 3 main dashboards (golden signals, app, DORA). Update frequency < 1s.

- [ ] **5.3 Grafana Loki (Logs Centralizados):**
  - **Deploy:** Via Helm chart `grafana/loki-stack`.
  - **Log Ingestion:**
    - Promtail (sidecar/DaemonSet) coleta stdout/stderr de pods.
    - Label strategy: `{namespace="production", pod="notification-service-abc", app="notification"}`.
  - **Retention:** 30d padrão, 7d para debug logs (cheaper storage tier).
  - **Query Language:** LogQL (similar a Prometheus).
    ```logql
    {namespace="production"} 
    | json 
    | level="error"
    ```
  - **Integration com Prometheus:** Loki + Prometheus queries lado a lado em Grafana.
  - **Critério:** Log queries < 1s mesmo com 30d dados. Alertas podem usar log patterns.

- [ ] **5.4 Jaeger (Tracing Distribuído):**
  - **Deploy (dev):** All-in-one Jaeger via Docker Compose (port 6831/udp, 16686/ui).
  - **Deploy (prod):** AWS X-Ray (managed, sem operação).
  - **Instrumentação (app):**
    - OpenTelemetry SDK (Go).
    - Exporter: `otlp/grpc` para Jaeger (dev), AWS X-Ray (prod).
    - Propagation: W3C Trace Context (headers HTTP).
    - Sampling: 10% (reduz custo, suficiente para debug).
  - **Spans:**
    - HTTP handler span (início → fim request).
    - DB query span (query + params).
    - SQS operation span (send/receive).
  - **Critério:** 100% das requisições com trace_id. Traces linkados entre serviços (futuros).

- [ ] **5.5 SLO & SLI:**
  - **SLO (Service Level Objectives):**
    - Availability: 99.95% (max 2h downtime/mês).
    - Latency p99: < 200ms.
    - Error rate: < 0.5%.
  - **SLI (Service Level Indicators):**
    - Availability SLI: `(success requests / total requests) * 100`.
    - Latency SLI: `histogram_quantile(0.99, ...)`.
    - Error SLI: `rate(errors_total[5m]) / rate(requests_total[5m])`.
  - **Dashboard SLO:** Burn rate (fast/slow), budget remaining, forecast.
  - **Alerting:** Alert se burn rate > 10x (significa 100 dias de budget em 10 dias).
  - **Critério:** SLO atualizado com base em dados reais. Alertas guiam prioritização.

- [ ] **5.6 Custom Metrics (App):**
  - **Métrica de negócio:**
    - `notifications_sent_total` – Total enviadas.
    - `notifications_sent_duration_seconds` – Histograma de tempo de envio.
    - `notification_delivery_status` – Gauge: pending, delivered, failed.
  - **Métrica de caos:**
    - `chaos_experiment_running` – Flag durante experimento.
    - `chaos_pod_kills_total` – Contagem de kills.
  - **Correlação:** Correlacionar métricas de negócio com caos para validar impacto.
  - **Critério:** Metrics expostas em Prometheus format. Sem cardinality explosion (evitar labels com user IDs).

---

## Fase 6: Resiliência e Chaos Engineering
**Objetivo:** Provar que sistema mantém SLO mesmo sob falhas. Validar rollback automático, observabilidade em crises.
**Plataforma:** LitmusChaos 3.x.

- [ ] **6.1 NetworkPolicy & Validação de Rede:**
  - **NetworkPolicy (K8s):**
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: notification-service-deny-all
    spec:
      podSelector:
        matchLabels:
          app: notification-service
      policyTypes:
      - Ingress
      - Egress
      ingress:
      - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
        ports:
        - protocol: TCP
          port: 8080
      egress:
      - to:
        - namespaceSelector: {}
        ports:
        - protocol: TCP
          port: 5432  # PostgreSQL
      - to:
        - namespaceSelector: {}
        ports:
        - protocol: TCP
          port: 443   # HTTPS (secrets API)
    ```
  - **Validação:**
    - `cilium policy audit` – Verifica regras efetivas.
    - Pod debug: `kubectl debug -it pod/app -- sh`, rodar `curl` para testar conectividade.
    - Ferramentas: `traceroute`, `tcpdump`, `netstat` dentro de pods.
  - **Teste:** Bloquear propositalmente um egress, validar que conexão falha + logs explicam (não é silent drop).
  - **Critério:** NetworkPolicy implementada, testada, sem deny-all silencioso.

- [ ] **6.2 Resiliência de Aplicação:**
  - **Circuit Breaker:** Para DB, SQS (ex: 5 falhas consecutivas → abrir por 30s).
  - **Retry Policy:** Exponential backoff (1s, 2s, 4s, 8s max). Máx 3 tentativas.
  - **Timeouts:** DB = 5s, SQS = 10s, HTTP = 30s.
  - **Graceful Shutdown:** 30s grace period. Termina requisições em flight, nega novas.
  - **Pool Sizes:** DB connections = 20 (max), SQS workers = 10.
  - **Critério:** App não trava. Logs claros de degradação. SQS retries não explodem.

- [ ] **6.3 LitmusChaos Setup:**
  - **Install:** Helm chart `litmuschaos/litmusctl` + CRDs.
  - **Namespace:** `litmus` com RBAC roles para injetar caos em `production`.
  - **RBAC:** SA `litmus` pode kill pods, exec commands (não root).
  - **Critério:** LitmusChaos operacional. Pode injetar falhas.

- [ ] **6.4 Experimentos de Falha:**

  **6.4.1 Pod Delete Chaos (Kill Instances):**
  - **Objetivo:** Validar que ReplicaSet reconstrói sem downtime.
  - **ChaosEngine:**
    ```yaml
    apiVersion: litmuschaos.io/v1alpha1
    kind: ChaosEngine
    metadata:
      name: pod-delete-exp
    spec:
      appinfo:
        appns: production
        applabel: "app=notification-service"
      engineState: "active"
      chaosServiceAccount: litmus
      experiments:
      - name: pod-delete
        spec:
          components:
            env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"
            - name: CHAOS_INTERVAL
              value: "30"
            - name: FORCE
              value: "true"
          probe:
          - name: "check-app-alive"
            type: "httpProbe"
            mode: "Continuous"
            httpProbe/inputs:
              url: "http://notification-service:8080/health/live"
              criteria: "=200"
              responseTimeout: "2"
    ```
  - **Duração:** 60s. Mata 1 pod a cada 30s.
  - **Probe:** HTTP `/health/live` deve responder 200 durante todo experimento.
  - **Validação:**
    - Pod recriado em < 10s (readiness probe).
    - RPS mantém. Error rate = 0% (requests failover).
    - Logs mostram restart event.
  - **Critério:** Zero downtime percebido. Métricas confirmam.

  **6.4.2 Network Latency Chaos (Slow Database):**
  - **Objetivo:** Validar que app degrada gracefully com DB lenta.
  - **ChaosEngine:**
    ```yaml
    apiVersion: litmuschaos.io/v1alpha1
    kind: ChaosEngine
    metadata:
      name: network-latency-exp
    spec:
      appinfo:
        appns: production
        applabel: "app=notification-service"
      experiments:
      - name: network-latency
        spec:
          components:
            env:
            - name: NETWORK_INTERFACE
              value: "eth0"
            - name: TC_LATENCY
              value: "250"  # 250ms
            - name: TOTAL_CHAOS_DURATION
              value: "120"
          probe:
          - name: "check-p99-latency"
            type: "promProbe"
            mode: "Edge"
            promProbe/inputs:
              endpoint: "http://prometheus:9090"
              query: "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[1m]))"
              threshold: "0.5"  # 500ms
              evaluationTimeout: "120"
    ```
  - **Latência:** 250ms adicionada a todo tráfego egress.
  - **Probe Prometheus:** p99 latência deve permanecer < 500ms (timeout principal é 30s, então 250ms + DB 250ms = ok).
  - **Validação:**
    - Error rate permanece < 0.5%.
    - CircuitBreaker não abre (< 5 erros).
    - SQS queue não acumula (workers conseguem processar).
  - **Critério:** App aguenta degradação sem falhar.

  **6.4.3 Disk Fill Chaos (Out of Space):**
  - **Objetivo:** Validar que app não trava se disco enche (ex: logs).
  - **ChaosEngine:** `disk-fill` experiment por 60s.
  - **Fill Size:** 80% da partição `/`.
  - **Validação:**
    - App continua respondendo.
    - Logs não ficam corrupt (write failures esperados).
    - Recovery pós-limpeza < 1min.
  - **Critério:** Graceful degradation. Alertas disparam.

  **6.4.4 CPU/Memory Stress:**
  - **Objetivo:** Validar limits e OOM behavior.
  - **ChaosEngine:** `pod-cpu-hog` + `pod-memory-hog`.
  - **Duration:** 60s. Stress até 80% de limite.
  - **Validation:**
    - Pod não mata (não atinge OOMKilled).
    - Requests slowdown previsível (não spike).
  - **Critério:** HPA escala se configured. Ou graceful degradation.

- [ ] **6.5 Análise & Rollback Automático:**
  - **Prometheus Alerts** durante caos:
    ```yaml
    - alert: HighErrorRateChaos
      expr: rate(http_requests_total{status=~"5.."}[1m]) > 0.01  # 1%
      for: 30s
      annotations:
        summary: "Error rate > 1% (possible chaos)"
    
    - alert: HighLatencyChaos
      expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[1m])) > 0.5
      for: 30s
    ```
  - **Argo Rollouts Analysis:**
    - Se canary em progresso E prometheus alert ativa → abort canary.
    - Rollback para versão anterior automático.
    - Notify Slack: "Canary aborted due to high error rate".
  - **Grafana:** Dashboard mostra timeline: chaos start → metrics spike → rollback triggered.
  - **Post-Mortem:** Saved query no Prometheus para análise ("Why did error rate spike?").
  - **Critério:** Rollback < 2min. Zero manual intervention.

- [ ] **6.6 Teste de Resiliência End-to-End:**
  - **Scenario:** Simultaneous chaos + canary deploy.
    1. Versão v1 rodando estável (N=2 pods).
    2. Canary deploy v2 começa (10% = 1 pod).
    3. Pod delete chaos mata v1 pod durante canary.
    4. Validar: Rollout continua, canary progride, zero user impact.
  - **Métrica de Sucesso:**
    - RPS continuou igual.
    - Error rate continuou < 0.5%.
    - p99 latência continuou < 200ms.
    - Sem human intervention.
  - **Critério:** Teste passa. Observabilidade explica cada falha.

- [ ] **6.7 Load Testing (K6/Gatling):**
  - **Setup:** K6 script simulando 100 RPS, 5min duration.
  - **Injetar caos** no meio do teste.
  - **Validar:** RPS mantém, erros < 0.5%, latência < SLO.
  - **Critério:** Load + chaos = controlado.