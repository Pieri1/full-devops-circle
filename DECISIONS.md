# DECISIONS.md – Decisões Tecnológicas & Justificativas

Este documento registra as decisões arquiteturais do projeto, justificativas e trade-offs. Referência para futuras discussões e onboarding.

---

## 1. Linguagem: Go 1.23

**Decisão:** Go (não Python, Rust, Node.js).

**Justificativa:**
- **Performance:** Binários compilados, latência p99 < 100ms (crítico para notificações).
- **Deployment:** Binário único (não runtime deps). Imagem distroless < 50MB.
- **Concorrência:** Goroutines leves (milhares). Ideal para processamento assíncrono (SQS consumer).
- **Standoff:** Tire de troca Python tem maior DX inicial, mas Go escala melhor em produção.
- **DevEx:** Toolchain Go é excepcional (go fmt, go test, go mod).

**Trade-offs:**
- ❌ Curva de aprendizado (interface, error handling explícito).
- ✅ Compilação única vs runtime flexibility.

**Quando Reconsiderar:**
- Se equipe é predominantemente Python (invest em async + FastAPI).
- Se prototipagem super-rápida é prioridade (Rust teria overhead maior).

---

## 2. Infraestrutura: Terraform + AWS + LocalStack

**Decisão:** Terraform (não CloudFormation, Pulumi, Ansible).

**Justificativa:**
- **Agnóstico:** Suporta AWS, GCP, Azure, Kubernetes (multi-cloud future-proof).
- **HCL:** Legível, declarativo, não Turing-complete (força constraints, segurança).
- **Comunidade:** Módulos reutilizáveis (Terraform Registry).
- **DevSecOps:** `tfsec`, OPA integrável em pipeline.

**Trade-offs:**
- ❌ State file complexo (S3 + DynamoDB para locking).
- ✅ Menor vendor lock-in.

**AWS (não GCP/Azure):**
- Mercado: 32% cloud share. Maior pool de talentos.
- Serviços: SQS, RDS PostgreSQL, Secrets Manager, X-Ray, EKS todos first-class.

**LocalStack (dev):**
- Custo: Dev não gasta $$ em AWS durante testes.
- Fidelidade: Comportamento SQS/S3 localmente similar (não perfeito, mas 95%).

**Quando Reconsiderar:**
- Pulumi: Se infra muito complexa ou multi-environment (Pulumi Automation API é superior).
- CloudFormation: Apenas se AWS-only forever (CloudFormation é verbose).

---

## 3. Orquestração: Kubernetes (EKS prod, K3s dev)

**Decisão:** Kubernetes (não Docker Swarm, Nomad, managed serverless).

**Justificativa:**
- **Padrão Indústria:** 90% enterprises. CNCF. Talent pool enorme.
- **Flexibilidade:** Suporta qualquer workload (stateless, stateful, batch, ML).
- **GitOps:** ArgoCD + Kustomize. Toda infra declarativa em git.
- **Multi-cloud:** Mesmo manifesto roda em EKS, GKE, AKS.

**K3s (dev):**
- Lightweight: Single binary. 300MB vs 1GB+ para full K8s.
- Fast: Startup ~1min vs ~5min kubeadm.
- Production-ready: Usado em prod (edge, IoT, labs).

**EKS (prod):**
- Managed: AWS gerencia control plane (redundância, patches).
- Security: IAM integration, VPC networking nativa.
- Cost: ~$73/month (control plane) + nodes. Alto, mas é preço de confiabilidade.

**Quando Reconsiderar:**
- Nomad: Se multi-environment (VMs + containers) necessário. HashiCorp stack.
- Serverless (Lambda): Apenas se 0 deployments/dia (cost inviável para sempre-on).
- Docker Swarm: Deprecated de facto. Evitar.

---

## 4. CI/CD: GitHub Actions + ArgoCD + Argo Rollouts

**Decisão:** GitHub Actions (não GitLab CI, Jenkins, CircleCI).

**Justificativa:**
- **Native:** GitHub Actions vem no GitHub (zero setup infra).
- **Pricing:** Generous free tier (3000 min/mês). Cheaper que enterprise CI.
- **DX:** YAML simples. Secrets integrado.

**ArgoCD (GitOps):**
- **Pull Model:** Cluster puxa mudanças (não push). Seguro, auditável.
- **Self-Healing:** Detecta drift, reconcilia automaticamente.
- **HA:** Rollback visual no UI. Não requer kubectl access.

**Argo Rollouts (Progressive Delivery):**
- **Canary:** 10% → 50% → 100% com análise automática.
- **Blue-Green:** Atomic cutover (não canary, instant swap).
- **Rollback:** Se p99 latência > threshold, rollback automático.

**Quando Reconsiderar:**
- GitLab CI: Se using GitLab repo (same ecosystem).
- Flux: Se GitOps puro (ArgoCD é mais "opinião").
- Spinnaker: Se multi-cloud deployments complexos.

---

## 5. Observabilidade: Prometheus + Grafana + Loki + Jaeger

**Decisão:** Open-source stack (não Datadog, New Relic, Splunk).

**Justificativa:**
- **Cost:** Prometheus/Grafana/Loki são free. Datadog é $$$$ em volume.
- **Flexibility:** Customizável. "Vendor lock-in" é seu próprio código.
- **CNCF:** Prometheus é standard de facto. Ecossistema rico.

**Prometheus:**
- **Metrics:** Time-series database. 15s scrape interval = resolution vs storage.
- **Alerting:** AlertManager + routing (Slack, PagerDuty).

**Grafana:**
- **Visualization:** Dashboards. Queries complex (Math, variable interpolation).
- **Alternative (Kibana):** Tightly coupled to Elasticsearch (overkill para metrics).

**Loki:**
- **Logs:** Efficient label-based indexing. Cheaper que ELK/Splunk.
- **Alternative (ELK):** More powerful, mas overkill + operational overhead.

**Jaeger (dev), X-Ray (prod):**
- **Jaeger:** All-in-one container. UI web. Perfeito para local testing.
- **X-Ray:** AWS managed. Zero operação. Correlação com CloudTrail/CloudWatch.

**Quando Reconsiderar:**
- Datadog: Se compliance requer (ex: PCI-DSS managed vendor).
- Elastic Stack: Se logs > 1TB/day (Loki pode não escalar).
- Tempo (Grafana): Se quiser logs + traces unificados em Grafana.

---

## 6. Segurança: Shift-Left (Semgrep + Trivy + Gitleaks)

**Decisão:** Scanning contínuo em dev/CI (não apenas scanning in prod).

**Justificativa:**
- **CISO mandate:** "Shift-left" = vulnerabilidades caras em prod, baratas em dev.
- **Feedback:** Dev vê results em 10s (CI passa em commit).
- **Tools:** Semgrep (SAST) + Trivy (image) + Gitleaks (secrets).

**Semgrep:**
- **Rule library:** OWASP, security patterns. Auto-updated.
- **Local:** `semgrep --config=p/security-audit` (fast, < 5s).

**Trivy:**
- **Comprehensive:** Escaneia Dockerfile, image layers, K8s manifests.
- **Integration:** Fácil em CI, SBOM output, container registry scanning.

**Gitleaks:**
- **Prevention:** Pre-commit hook bloqueia secrets. CI double-check.
- **Alternative (TruffleHog):** Ambos complementares. Gitleaks é mais rápido.

**OPA/Gatekeeper:**
- **PolicyAsCode:** K8s admission controller. Rejeita pods non-compliant.
- **Policy:** Non-root, resource limits, image pull policy.

**Quando Reconsiderar:**
- Snyk: Se dependências críticas (SCA avançado). Pode ser usado in tandem.
- SonarQube: Se análise super-granular necessária. Enterprise-grade.

---

## 7. FinOps: Infracost + Kubecost + AWS Tagging

**Decisão:** Cost tracking transparente em todo pipeline.

**Justificativa:**
- **CFO mandate:** Custo deve ser atributo de first-class (não afterthought).
- **Tools:** Infracost (IaC), Kubecost (runtime), AWS Budget Alerts.

**Infracost:**
- **CI Integration:** Comenta em PR com delta ($X/month).
- **Guardrail:** Reject PRs se delta > threshold.

**Kubecost:**
- **K8s Native:** Reads requests/limits + AWS pricing. Custo por pod.
- **Attribution:** Tags (cost-center, app) = chargeback preciso.

**AWS Tagging:**
- **Mandatory:** Tags obrigatórias (env, cost-center, owner).
- **Enforcement:** AWS Config rules. Non-compliant resources = auto-terminate.

**Quando Reconsiderar:**
- Vantage/CloudZero: Se FinOps super-avançado necessário (anomaly detection).

---

## 8. Resiliência: LitmusChaos + Argo Rollouts Automated Rollback

**Decisão:** Chaos engineering integrado (não post-deploy testing).

**Justificativa:**
- **Confidence:** Prova que sistema aguenta falhas reais.
- **Automation:** Rollouts rollback automático baseado em métricas.

**LitmusChaos:**
- **Operators:** Pod delete, network latency, disk fill, CPU stress.
- **Probes:** Valida health durante caos (HTTP, prometheus queries).

**Argo Rollouts:**
- **Analysis:** Prometheus query como "analysis gate" durante canary.
- **Automatic Rollback:** Se p99 latência > 500ms OU error rate > 1% → abort.

**When Reconsider:**
- Chaos Mesh: Mais overhead, resource-intensive. LitmusChaos é lighter.
- NetworkPolicy Chaos: Complementar (Cilium plugin).

---

## 9. Versionamento de API: /v1/ Path (não Accept Header)

**Decisão:** Path-based versioning (`/v1/`, `/v2/`).

**Justificativa:**
- **Debuggability:** URL é clara (logs, curl, browser).
- **Cache:** HTTP caches respeitam paths (proxies, CDN).
- **Analytics:** Easy track por versão (logs = v1 vs v2 ratio).

**Estratégia de Deprecação:**
- Manter v1 + v2 em paralelo por 6 meses.
- Response header: `Deprecation: true`, `Sunset: 2026-09-19`.
- PagerDuty alert se v1 ainda tem traffic em final month.

**Quando Reconsiderar:**
- Accept: Se API é consumida via SDK (versioning em SDK headers).
- Subdomain: Se multi-tenant (api-v1.example.com vs api-v2.example.com).

---

## 10. Database: PostgreSQL (não MongoDB, DynamoDB, Redis)

**Decisão:** PostgreSQL (RDS managed).

**Justificativa:**
- **ACID:** Transações. Data consistency (vs eventual consistency NoSQL).
- **Query Power:** SQL is powerful. JOINs, aggregations.
- **Maintenance:** RDS automates backups, patches, failover.

**Trade-offs:**
- ❌ Scaling: Vertical scaling até ~db.r7g.16xlarge. Sharding é pain.
- ✅ Data integrity (critical para notifications = não pode perder).

**When Consider NoSQL:**
- DynamoDB: Se schema evolui rápido + sharding automático necessário (serverless).
- MongoDB: Se documents nested (rich schema). RDS + JSONB quase tão bom.
- Redis: Caching layer (em frente de PostgreSQL), não primary store.

---

## 11. Message Queue: AWS SQS (não RabbitMQ, Kafka)

**Decisão:** SQS (managed).

**Justificativa:**
- **Simplicity:** Send, receive, delete. Sem operação.
- **Scaling:** Automático. Unlimited throughput.
- **Integration:** Lambda, SNS fanout, DLQ built-in.

**Trade-offs:**
- ❌ Message ordering não garantido (FIFO queue disponível, mas mais lento).
- ❌ Max message size = 256KB (vs Kafka unlimited).
- ✅ Custo baixo (~$0.4 por 1M requests).

**When Consider:**
- Kafka: Se stream processing (complex topology: filters, joins, windowing).
- RabbitMQ: Se on-prem preferred. Advanced routing (topic exchange).

---

## 12. Secrets Management: AWS Secrets Manager + External Secrets Operator

**Decisão:** AWS Secrets Manager (prod) + Sealed Secrets (K8s).

**Justificativa:**
- **Encryption:** AES-256. Key rotation automatic q90d.
- **Audit:** CloudTrail logs all access.
- **Scaling:** Managed. No operational overhead.

**External Secrets Operator:**
- **Sync:** Secrets from AWS → K8s Secrets.
- **Refresh:** Auto-sync on rotation. App receives updated secret in cache.

**Local (dev):**
- `.env.local` (gitignored). Testcontainers injeta no container.
- Não usar LocalStack Secrets Manager (overhead).

**When Consider:**
- HashiCorp Vault: Se multi-cloud secrets needed. Overkill para AWS-only.

---

## 13. Disaster Recovery: RPO = 1 day, RTO = 2 hours

**Decisão:** Daily backups. Restore time target 2h.

**Justificativa:**
- **RPO:** 1 dia de dados loss é aceitável (notifications não critical).
- **RTO:** 2h restore = produção back online em manhã.
- **Cost:** Point-in-time recovery 30 dias (AWS standard).

**Backup Strategy:**
- RDS: Automated snapshots daily. Retained 30 days.
- Terraform State: S3 versioning + Glacier (disaster recovery).
- Database: Tested restore q30d (valida backup integrity).

**When Reconsider:**
- RPO < 1 hour: Se transações críticas (bank, healthcare).
- RTO < 30 min: Se SLA exigir. Requer multi-region active-active.

---

## 14. Environment Strategy: local, dev, staging, prod

**Decisão:** 4 ambientes (não 2 ou 5).

**Justificativa:**
- **Local:** Dev container. K3s + LocalStack. Rápido feedback (< 2s).
- **Dev:** AWS dev. Small instances. For integration testing. Reset daily.
- **Staging:** AWS staging. Prod-like. Before production deploy.
- **Prod:** AWS prod. HA, multi-AZ. Canary deploy.

**Promotion Path:** local → dev → staging → prod (unidirectional).

**Data:**
- Local/Dev: Seed data (anonymized).
- Staging: Snapshot de prod (masked PII). Realistic volume.
- Prod: Real user data (encrypted).

---

## Summary Table

| Decisão | Escolha | Alternativa Rejeitada | Razão Principal |
|---------|---------|----------------------|-----------------|
| Linguagem | Go | Python, Rust, Node | Latência baixa, binário único, concorrência |
| IaC | Terraform | CloudFormation, Pulumi | Agnóstico, HCL legível, comunidade |
| Nuvem | AWS | GCP, Azure | Market share, serviços first-class |
| Dev Cloud | LocalStack | Sem mock | Custo dev, fidelidade 95% |
| Container Orch | K8s | Docker Swarm, Nomad | Padrão indústria, CNCF, multi-cloud |
| CI/CD | GitHub Actions | GitLab CI, Jenkins | Native, pricing generous |
| GitOps | ArgoCD | Flux | UI web, rollback visual, HA |
| Progressive Deploy | Argo Rollouts | Blue-Green | Canary análise automática |
| Metrics | Prometheus | Datadog | Open-source, CNCF, cost |
| Logs | Loki | ELK, Splunk | Efficient labels, cost |
| Traces | Jaeger (dev), X-Ray (prod) | Tempo, Honeycomb | Local flexibility + managed prod |
| SAST | Semgrep | SonarQube | Rápido, regras atualizadas |
| Image Scan | Trivy | Clair | Comprehensive, SBOM |
| Secrets Det | Gitleaks | TruffleHog | Rápido, pre-commit |
| PolicyAsCode | OPA/Gatekeeper | Kyverno | Flexible, CNCF |
| FinOps | Infracost + Kubecost | Vantage | Transparent, integrado |
| Chaos Eng | LitmusChaos | Chaos Mesh | Lighter, CNCF |
| API Version | Path-based (/v1/) | Accept header | Debug, caching, analytics |
| Database | PostgreSQL | MongoDB, DynamoDB | ACID, SQL power, integrity |
| Queue | SQS | RabbitMQ, Kafka | Managed, scaling, AWS integration |
| Secrets | AWS Secrets Manager | HashiCorp Vault | Encryption, rotation, audit |
| DR | RPO=1d, RTO=2h | RPO<1h, RTO<30m | Cost-effective para use case |
| Ambientes | 4 (local, dev, staging, prod) | 2 ou 5 | Balance: feedback rápido vs prod-like |

