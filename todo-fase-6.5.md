# Fase 6.5: Governance, Compliance & Operational Excellence

**Objetivo:** Garantir que todo mudança é rastreável, aprovável, auditável. Compliance com regulações (ex: LGPD, SOX). FinOps controlado.

---

## Governance & Approval Gating

- [ ] **6.5.1 Manual Approval para Produção:**
  - **Policy:** Toda mudança em `prod` requer aprovação manual de 1 reviewer (não-author) + 1 platform engineer.
  - **GitHub:** Branch protection rules em `app-config/main`:
    - Require 2 PR reviews.
    - Require CODEOWNERS approval (`.github/CODEOWNERS` designa platform team).
    - Require status checks pass (CI green).
  - **Delay:** Mínimo 1h entre approval e merge (janela de reflexão).
  - **Timeout:** Se ninguém aprova em 24h, PR requer "refresh" (re-run CI).
  - **Critério:** Zero prod deploys sem rastreabilidade. Audit log no GitHub.

- [ ] **6.5.2 Change Advisory Board (CAB) Lightweight:**
  - **Mudanças Críticas:** Database schema changes, security policy changes, cost > $500/month.
  - **Processo:** 
    1. Dev abre issue (tipo: `[CAB] ...`).
    2. Notifica Slack #eng-platform.
    3. Platform engineer revisa em < 24h.
    4. Simula impacto (staging).
    5. Aprova ou pede ajustes.
  - **Risk Assessment:** Impacto (data loss?, downtime?), rollback plan (quão rápido?).
  - **Critério:** Nenhuma surpresa em prod.

- [ ] **6.5.3 Rollback Plan Mandatório:**
  - **Template no PR:**
    ```markdown
    ## Rollback Plan
    - [ ] Tested: Rollback em staging sucedeu?
    - [ ] Duration: Rollback time < X minutos
    - [ ] Data: Se mudança de schema, passos de downgrade?
    - [ ] Communication: Quando notificar stakeholders?
    ```
  - **Cenários:**
    - **App code:** Imediato (Argo Rollouts revert).
    - **Database schema:** Forward-compatible (add column, não remove). Sempre reversível.
    - **Config:** Revert em ArgoCD (git revert).
  - **Critério:** Todo PR descreve como desfazer.

---

## Compliance & Audit

- [ ] **6.5.4 Auditoria de Acesso (RBAC & Logging):**
  - **K8s RBAC:** 
    - `developers` role: `get`, `list` pods, logs. Sem `delete`, `edit`.
    - `platform-team` role: `*` (admin).
    - `ci-cd` service account: Específico (create `Rollout`, update image tag).
  - **CloudTrail (AWS):**
    - Log todas API calls (S3, Secrets Manager, EKS).
    - Retenção: 90 dias. Archivado em S3 Glacier 1 ano.
  - **Kubernetes Audit Log:**
    - Log todos `create`, `delete`, `patch` (spec changes).
    - Enviado para CloudWatch Logs.
    - Query: "Quem deletou pod X em T0?"
  - **Critério:** 100% rastreabilidade. Zero ações anônimas.

- [ ] **6.5.5 Secret Rotation & Compliance:**
  - **Política:** Secrets rotacionam q90d (automático).
  - **Validation:** Pós-rotação, app continua funcional (não requer restart).
  - **Logging:** CloudTrail logs rotação (quem, quando, sucesso/falha).
  - **LGPD Compliance:** 
    - PII (dados pessoais) nunca em logs. Mascarar em Loki se escapar.
    - Tenho direito de apagar? Setup DynamoDB TTL para dados temporários.
  - **Critério:** Secrets nunca vazam. Compliance check passa.

- [ ] **6.5.6 Data Retention & Backup:**
  - **Database:** Daily backup (AWS RDS automated). Retention 30 dias. Point-in-time recovery suportado.
  - **Logs:** 
    - Hot: 30 dias (Prometheus, Loki).
    - Warm: 1 ano (S3 Standard-IA).
    - Cold: Indefinido (Glacier).
  - **State Terraform:** Snapshots daily em Glacier (disaster recovery).
  - **Testing:** Restore do backup q mensal (valida backup integrity).
  - **Critério:** RPO = 1 dia, RTO = 2h (objetivo).

---

## FinOps Avançado

- [ ] **6.5.7 Tagging & Cost Allocation:**
  - **Tags obrigatórias (AWS):**
    - `Environment` = dev/staging/prod.
    - `CostCenter` = eng/infra/product.
    - `Application` = notification-service.
    - `Owner` = team name.
    - `ChargebackCode` = para billing interno.
  - **Enforcement:** Tag policy via AWS Config. Resources sem tags = terminate após 7 dias aviso.
  - **Kubecost Labels (K8s):**
    - `cost-center`, `app`, `env` (maps to AWS tags).
    - Dashboard Kubecost por label = alocação precisa.
  - **Critério:** CFO consegue rastrear gasto por aplicação/time.

- [ ] **6.5.8 Budget Alerts & Anomaly Detection:**
  - **Budget Setup (AWS Budgets):**
    - Dev: $500/month. Alert @ 80%, 100%.
    - Staging: $200/month.
    - Prod: $2000/month (tuned based on usage).
  - **Anomaly Detection:** 
    - CloudWatch Anomaly Detector em EC2/RDS costs.
    - Alert se custo 25% acima baseline.
    - Trigger investigation (ex: auto-scaling descontrolado?).
  - **Infracost Guardrail:** Rejeitar PRs se delta > $200/month (prod) ou $50/month (dev).
  - **Critério:** Sem surpresas na fatura. Waste detectado em horas.

- [ ] **6.5.9 Resource Optimization:**
  - **Right-Sizing:**
    - Prometheus job: `job_rds_oversized`. Se CPU < 20% por 30d → downsize.
    - Slack notification: "RDS pode downsize de db.t4g.large → db.t4g.small (-$X/month)".
  - **Spot Instances (Dev):**
    - EKS node group 50% spot (savings ~80%).
    - App tolera disruption (readiness probe fail = evict gracefully).
  - **Storage Cleanup:**
    - ECR: Delete untagged images após 7d.
    - S3: Lifecycle rule → Glacier após 90d.
  - **Critério:** Cost/month decreasing 5% quarterly.

---

## Operational Excellence

- [ ] **6.5.10 On-Call & Incident Response:**
  - **On-Call Rotation:** 1 engineer per week. PagerDuty rules:
    - Critical alert → page.
    - Warning alert → Slack only.
  - **Incident Response:**
    1. Alert dispara → PagerDuty notifica on-call.
    2. On-call verifica Grafana (golden signals). Se clear → dismiss.
    3. Se real issue → open incident (Slack thread + incident tracker).
    4. Post-incident review (24h) → action items.
  - **Documentation:** 
    - Runbook por alerta (ex: "HighErrorRate" → check logs, check recent deploy, rollback?).
    - Playbook em `docs/runbooks/`.
  - **Critério:** MTTR < 30min. Respostas automáticas reduzem page 50%.

- [ ] **6.5.11 SLO Tracking & Blameless Culture:**
  - **SLO Burn Rate:**
    - Track burn rate por semana.
    - Slow burn (< 1x/week) = business as usual.
    - Fast burn (> 10x/week) = freeze features, focus on reliability.
  - **Blameless Postmortem:**
    - Não culpar pessoas. Culpar processos.
    - Exemplo ruim: "Engineer merged without testing".
    - Exemplo bom: "Teste de integração não rodou em CI (bug em fixture)".
  - **Retrospective:** Quem, O quê, Quando, Por que, Como evitar.
  - **Critério:** SLO violations documentadas. Ações preventivas rastreadas.

- [ ] **6.5.12 Disaster Recovery Drill (q trimestral):**
  - **Cenários:**
    1. Database corruption → restore from backup.
    2. Complete cluster failure → rebuild from IaC.
    3. Secrets Manager unavailable → app fallback behavior.
  - **Execution:**
    - Não anunciado (teste de prontidão).
    - Simulado em ambiente de staging.
    - Medido: Tempo para detectar + recover.
  - **Resultado:** RTO < objetivo (ex: 2h).
  - **Critério:** Zero surpresa. Runbooks atualizadas após cada drill.

- [ ] **6.5.13 Dependency Management & Vulnerability Tracking:**
  - **Tool:** Dependabot (GitHub) + Snyk para deps.
    - Scans diário.
    - Auto-PR para patch versions (sem mudança breaking).
    - Manual review para minor/major.
  - **Policy:**
    - Critical: Fix em 7 dias.
    - High: Fix em 30 dias.
    - Medium: Fix em 60 dias.
  - **Go mod tidy:** CI roda `go mod tidy -compat=1.23` para depuração.
  - **Critério:** Nenhuma dep critical/high sem plano.

---

## Security Scanning (SAST/SCA/DAST)

- [ ] **6.5.14 Continuous Security Scanning:**
  - **SAST (Semgrep):** Roda em commit.
  - **SCA (Trivy/Snyk):** Roda em commit (deps) + image build.
  - **DAST (OWASP ZAP):** Roda em staging nightly.
    - GET /health/live (deve ser 200).
    - POST /v1/notifications com payloads suspeitos (ex: SQL injection, XXE).
  - **Supply Chain Security:** Sigstore (Cosign) assina + valida imagens.
  - **Critério:** Zero high/critical findings em main branch.

- [ ] **6.5.15 Secrets Detection (Pré-Produção):**
  - **Gitleaks:** Bloqueia hardcoded secrets.
  - **TruffleHog:** Histórico git (detecta secrets já commitados, mesmo se deletados).
    - Roda em CI se PR incluir commits antigos.
  - **Remediation:** Se secret vazado → rotate immediately, audit logs.
  - **Critério:** Nenhuma secret em repositório. Histórico limpo.

---

## Summary: Phase 6.5 Success Criteria

| Aspecto | Critério | Verificação |
|---------|----------|-------------|
| **Governance** | Todas mudanças prod com aprovação + rastreamento | GitHub audit log, CAB log |
| **Compliance** | 100% rastreabilidade. PII masked. Backup testado. | CloudTrail, DynamoDB TTL, restore drill |
| **FinOps** | Custo previsível. Waste detectado < 1h. Trending down 5%/q | Budget alerts, anomaly detection, cost/month trend |
| **Operações** | MTTR < 30min. SLO > target. Zero surpresa em incident. | PagerDuty metrics, SLO dashboard, postmortem review |
| **Security** | Findings scanned. Deps updated. Secrets protected. | Trivy/Semgrep/Snyk clean, Gitleaks + TruffleHog pass |

