# Full DevOps Circle – Microserviço de Notificações

Projeto de referência com **stack completo** de DevOps, DevEx, GitOps, Observabilidade e Engenharia de Caos. Foco em tendências de mercado, automação total e zero downtime.

---

## 🎯 Objetivo

Construir uma arquitetura **robusta, observável e resiliente** para um serviço de notificações, demonstrando:

- ✅ **DevEx:** Setup < 10min. Desenvolvimento totalmente containerizado (Dev Container).
- ✅ **IaC:** 100% Terraform. Modular, testável, custo controlado.
- ✅ **CI/CD:** GitHub Actions → ArgoCD (GitOps). Canary deploy automático.
- ✅ **Observabilidade:** Prometheus + Grafana + Loki + Jaeger. Golden Signals + DORA Metrics.
- ✅ **Segurança:** Shift-left (Semgrep, Gitleaks, Trivy). Secrets gerenciados. OPA/Gatekeeper.
- ✅ **Resiliência:** LitmusChaos + Argo Rollouts. Rollback automático baseado em métricas.
- ✅ **Governance:** Auditoria, compliance, FinOps, on-call, disaster recovery.

---

## 📚 Documentação

| Documento | Propósito |
|-----------|----------|
| **[context.md](./context.md)** | Stack tecnológico, decisões de design, workflow completo. |
| **[todo.md](./todo.md)** | Checklist detalhado com 6 fases + critérios de aceitação. |
| **[todo-fase-6.5.md](./todo-fase-6.5.md)** | Fase 6.5: Governance, compliance, FinOps, disaster recovery. |
| **[DECISIONS.md](./DECISIONS.md)** | Justificativas para cada escolha tecnológica + trade-offs. |
| **[ARCHITECTURE.md](./ARCHITECTURE.md)** | Diagramas e fluxos de todo sistema (app, K8s, CI/CD, observabilidade). |

---

## 🏗️ Arquitetura (Alto Nível)

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   GitHub    │       │  GitHub     │       │    AWS      │
│   (code)    │──────→│  Actions    │──────→│   (ECR)     │
└─────────────┘       │  (CI)       │       └─────────────┘
                      └──────┬──────┘              │
                             │                    ↓
                             ↓          ┌──────────────────┐
                      ┌─────────────┐  │   AWS EKS        │
                      │  ArgoCD     │  │  (K8s Cluster)   │
                      │  (GitOps)   │  │                  │
                      └──────┬──────┘  │ • Notification   │
                             │        │   Service        │
                      ┌──────v──────┐  │ • Prometheus     │
                      │   app-config│  │ • Grafana        │
                      │   (git repo)│  │ • Loki           │
                      └─────────────┘  │ • ArgoCD         │
                                       │ • LitmusChaos    │
                                       └────────┬─────────┘
                                                │
                                    ┌───────────┴──────────┐
                                    ↓                      ↓
                          ┌──────────────────┐  ┌────────────────┐
                          │   RDS Postgres   │  │   AWS SQS      │
                          │  (Persistence)   │  │  (Messaging)   │
                          └──────────────────┘  └────────────────┘
```

---

## 🚀 Tech Stack

### Aplicação & DevEx
- **Linguagem:** Go 1.23 (binários estáticos, latência < 100ms)
- **Desenvolvimento:** Dev Container (VS Code)
- **Automação:** Taskfile.yml (task up, task test, task lint)
- **Documentação:** OpenAPI + Mermaid

### IaC & Cloud
- **Provisionador:** Terraform 1.6+ (módulos reutilizáveis)
- **Nuvem:** AWS EKS (prod) + LocalStack (dev)
- **Orquestração:** Kubernetes (K3s local, EKS prod)
- **Secrets:** AWS Secrets Manager + External Secrets Operator

### CI/CD & GitOps
- **CI:** GitHub Actions (lint, test, scan, build, push)
- **Repositório Manifestos:** Separado (`app-config`)
- **GitOps:** ArgoCD (pull model, self-healing)
- **Progressive Deploy:** Argo Rollouts (Canary 10%→50%→100%)

### Segurança (DevSecOps)
- **SAST:** Semgrep (OWASP patterns)
- **SCA:** Trivy (imagens, configs), Snyk (deps)
- **Secrets:** Gitleaks (pre-commit), TruffleHog (histórico)
- **PolicyAsCode:** OPA/Gatekeeper (non-root, limits, signed images)
- **SBOM:** Syft
- **Signing:** Cosign (image signing + OIDC)

### Observabilidade
- **Métricas:** Prometheus (15s scrape, 30d retention)
- **Visualização:** Grafana (dashboards: Golden Signals, DORA, SLO)
- **Logs:** Loki (label-based, 30d hot + S3 warm)
- **Traces:** Jaeger (dev), AWS X-Ray (prod)
- **Alerting:** AlertManager → Slack + PagerDuty

### Resiliência
- **Chaos Engineering:** LitmusChaos (pod delete, network latency, CPU/disk stress)
- **Auto Rollback:** Argo Rollouts com análise Prometheus
- **Circuit Breaker:** App-level (DB, SQS)
- **Graceful Shutdown:** 30s grace period

### Governance
- **Approval Gate:** 2x reviews + platform engineer para prod
- **Auditoria:** CloudTrail + K8s audit logs
- **SLO:** 99.95% availability, p99 < 200ms
- **FinOps:** Infracost (PR comment), Kubecost (runtime tracking)
- **Disaster Recovery:** Daily backups, restore testado q30d

---

## 📋 Fases do Projeto

### Fase 1: DevEx & Fundação
- Dev Container com Go, Terraform, K3s
- Taskfile para automação
- LocalStack para simulação AWS
- Documentação viva (Mermaid)

### Fase 2: Aplicação & Shift-Left Security
- API REST (Go, standard layout)
- Health checks (liveness, readiness)
- Testes com Testcontainers
- SAST (Semgrep), Gitleaks, Dockerfile otimizado

### Fase 3: IaC & FinOps
- Módulos Terraform (VPC, EKS, RDS, Secrets)
- Ambientes declarativos (local, dev, staging, prod)
- Validação (tfsec, Trivy, terraform validate)
- Infracost para FinOps

### Fase 4: CI/CD & GitOps
- Pipeline GitHub Actions (lint, test, build, scan)
- Repositório de manifestos separado
- ArgoCD com sincronização automática
- Argo Rollouts com canary automático + rollback

### Fase 5: Observabilidade
- Stack Prometheus + Grafana + Loki + Jaeger
- Golden Signals dashboard
- DORA Metrics (lead time, deployment frequency)
- SLO/SLI tracking, alerting

### Fase 6: Resiliência & Chaos Engineering
- NetworkPolicy + validação de rede
- Experimentos de falha (pod delete, latency, disk fill, CPU)
- Análise & rollback automático
- Load testing + chaos simultâneos

### Fase 6.5: Governance & Compliance
- Approval gates para produção
- Auditoria completa (CloudTrail, K8s logs)
- Change Advisory Board (CAB)
- Disaster recovery drills (q3m)
- FinOps avançado (tagging, anomaly detection)
- On-call + incident response
- Blameless postmortem

---

## 🎯 Decisões Tecnológicas (Resumidas)

| Aspecto | Escolha | Por Quê |
|--------|--------|---------|
| Linguagem | **Go 1.23** | Latência baixa, binários estáticos, concorrência |
| IaC | **Terraform** | Agnóstico, HCL, comunidade, modules |
| Nuvem | **AWS** | Market leader (32%), talent pool, serviços |
| Orquestração | **Kubernetes** | Padrão CNCF, multi-cloud, flexibilidade |
| GitOps | **ArgoCD** | UI web, rollback visual, HA |
| Progressive | **Argo Rollouts** | Canary automático + análise |
| Métricas | **Prometheus** | Open-source, CNCF, custo previsível |
| Logs | **Loki** | Escalável, label-based, cheap |
| Traces | **Jaeger/X-Ray** | Jaeger dev (fast), X-Ray prod (managed) |
| SAST | **Semgrep** | Rápido, rules atualizadas, local run |
| Scan Imagem | **Trivy** | Comprehensive, SBOM, registry integration |
| PolicyAsCode | **OPA/Gatekeeper** | Flexible, CNCF, integra ArgoCD |
| FinOps | **Infracost + Kubecost** | Transparent, automático |
| Chaos | **LitmusChaos** | Leve, CNCF, Argo Rollouts integration |

---

## 🔐 Segurança

**7 Camadas de Defesa:**
1. **Shift-Left:** Gitleaks (pre-commit), Semgrep, Trivy em CI
2. **Build:** Docker multi-stage, distroless, non-root
3. **Registry:** Trivy scan, Cosign signing, SBOM
4. **Admission:** OPA/Gatekeeper (non-root, limits, signed images)
5. **Network:** NetworkPolicy (deny-all default, allow específico)
6. **Runtime:** Circuit breaker, rate limiting, input validation
7. **Audit:** CloudTrail, K8s audit logs, Gitleaks + TruffleHog

---

## 📊 Observabilidade

**Golden Signals Dashboard:**
- Latência: p50, p95, p99
- Tráfego: RPS (requests/sec)
- Erros: Error rate %
- Saturação: CPU, Memory %

**DORA Metrics:**
- Lead Time for Changes
- Deployment Frequency
- Mean Time to Recovery (MTTR)
- Change Failure Rate

---

## 🔥 Resiliência

**Padrões Implementados:**
- Circuit Breaker (DB, SQS)
- Retry com exponential backoff
- Timeout (HTTP 30s, DB 5s, SQS 10s)
- Graceful degradation
- Pod restarts (readiness probe)
- HPA (horizontal scaling)

**Proof by Chaos:**
- Pod Delete Chaos (zero downtime)
- Network Latency Chaos (graceful degradation)
- Disk Fill Chaos (error handling)
- CPU Stress Chaos (scaling)

---

## 💰 FinOps

- **Infracost:** Delta de custo em PRs (rejeita se > threshold)
- **Kubecost:** Custo por namespace/pod/app (chargeback)
- **AWS Budgets:** Alertas por env (dev $500/m, prod $2k/m)
- **Anomaly Detection:** Alert se custo 25% acima baseline
- **Right-Sizing:** Recomendações automáticas

---

## 🎓 Como Usar

1. **Leia [context.md](./context.md)** para entender o projeto
2. **Consulte [DECISIONS.md](./DECISIONS.md)** para justificativas
3. **Siga [todo.md](./todo.md)** fase por fase
4. **Revise [ARCHITECTURE.md](./ARCHITECTURE.md)** para fluxos
5. **Implemente [todo-fase-6.5.md](./todo-fase-6.5.md)** para governance

---

## 📞 Contato

Dúvidas sobre:
- **DevOps/IaC:** Terraform, K8s, ArgoCD
- **Observabilidade:** Prometheus, Grafana, Loki
- **Segurança:** Shift-left, OPA, secrets
- **Chaos Engineering:** LitmusChaos, resilience patterns

Veja [DECISIONS.md](./DECISIONS.md) para "Quando Reconsiderar" cada tecnologia.

---

## 📄 Licença

MIT (Documentação educacional)

---

**Versão:** 1.0 (2026-03-19)  
**Status:** Referência de melhores práticas DevOps/DevEx
