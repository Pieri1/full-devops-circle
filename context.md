### `context.md`

```markdown
# Contexto do Projeto: Ciclo Completo DevOps & DevEx

## Objetivo do Projeto
Construir uma infraestrutura robusta e automatizada para um microserviço de **Notificações**, focando em Developer Experience (DevEx), automação de infraestrutura (IaC), entrega contínua (GitOps) e observabilidade avançada. O projeto visa simular um ambiente de produção real com resiliência e segurança.

---

## Stack Tecnológica

### 1. Aplicação & DevEx
- **Linguagem:** Go 1.23+ – Binários estáticos, baixa latência (< 100ms), startup instantâneo. Trade-off: curva de aprendizado maior que Python, mas essencial para resiliência e custos de infraestrutura.
- **Ambiente Local:** Dev Containers (VS Code + Docker) – Funciona em Linux, macOS e Windows (WSL2). Consistent com produção (mesma imagem, mesmo OS).
- **Automação de CLI:** `Taskfile.yml` – Mais moderno que Makefile, polyglot, melhor DX.
- **Documentação:** Swagger/OpenAPI + Mermaid.js para diagramas de arquitetura. Hospedado em `/docs` via Swagger UI no serviço.

### 2. Infraestrutura como Código (IaC) & Cloud
- **Provisionamento:** Terraform 1.6+ – Módulos reutilizáveis, `terraform test` para validação, estado em S3 + DynamoDB com locking.
- **Ambiente de Nuvem:** AWS (prod) + LocalStack (local) – SQS, RDS PostgreSQL, S3, Secrets Manager. LocalStack reduz custos de desenvolvimento.
- **Orquestração:** Kubernetes – EKS (prod) + K3s em Docker (dev). K3s leve e rápido para validar manifestos localmente.
- **Secrets:** AWS Secrets Manager (prod) + Sealed Secrets Operator (K8s) ou External Secrets Operator para sincronizar chaves.

### 3. CI/CD & GitOps
- **Pipeline de CI:** GitHub Actions – Lint, testes unitários com `testcontainers-go`, scan com Trivy, SBOM com Syft, image signing com Cosign.
- **Repositório de Manifestos:** Separado (GitOps pattern) – CI atualiza tags de imagem, ArgoCD sincroniza automaticamente (*self-healing*).
- **Entrega (CD):** ArgoCD (latest) – Sincronização automática com validação de PolicyAsCode (OPA/Gatekeeper).
- **Estratégia de Deploy:** Argo Rollouts com Canary (10% → pausa 5min → 50% → 100%) + rollback automático baseado em alertas Prometheus.

### 4. Segurança (DevSecOps)
- **SAST/SCA:** Semgrep (app), Trivy (imagens + IaC), tfsec (Terraform).
- **Secrets Prevention:** Gitleaks em pre-commit hook + pipeline GitHub Actions.
- **Image Signing:** Cosign para assinatura de imagens, validação em Kubernetes via PolicyAsCode.
- **FinOps:** Infracost em PRs, Kubecost em produção para accountability de custos por namespace/app.
- **Compliance:** OPA/Gatekeeper para PolicyAsCode, auditoria com Falco para detecção de anomalias em runtime.

### 5. Observabilidade & Resiliência
- **Métricas:** Prometheus (scrape 15s) + Grafana – Golden Signals (latência, tráfego, erros, saturação) + DORA Metrics (lead time, deployment frequency).
- **Logs:** Grafana Loki com label strategy (namespace, pod, app). Retenção: 30d padrão, 7d para logs de debug.
- **Tracing:** OpenTelemetry SDK (Go) → Jaeger para dev/staging, AWS X-Ray para prod.
- **Alerting:** Prometheus AlertManager + Webhook para Slack/PagerDuty. SLO: 99.95% availability, p99 < 200ms.
- **Engenharia de Caos:** LitmusChaos (pod delete, network latency, disk fill, CPU stress).

---

## Estrutura de Diretórios Proposta
```text
.
├── .devcontainer/        # Configuração do ambiente de desenvolvimento
├── .github/              # Workflows do GitHub Actions (CI)
├── app/                  # Código fonte do microserviço
├── infra/
│   ├── terraform/        # Módulos de VPC, EKS, RDS
│   └── k8s/              # Manifestos ou Helm Charts
├── gitops/               # Configurações do ArgoCD e Rollouts
├── monitoring/           # Dashboards do Grafana e regras do Prometheus
└── Taskfile.yml          # Orquestrador de comandos do projeto
```

---

## Fluxo de Trabalho (Workflow)

### Dev Loop Local
1. Dev clona e executa `task up` → Dev Container + LocalStack + K3s inicializam.
2. Dev muda código → Lint automático, tests, Gitleaks (pre-commit).
3. Dev commita → GitHub Actions dispara CI.

### CI/CD Pipeline
4. **CI (GitHub Actions):** Lint, UT + integration tests (Testcontainers), SAST (Semgrep), build image, Trivy scan, SBOM (Syft), sign image (Cosign).
5. **FinOps:** Infracost comenta no PR com delta de custo (Terraform changes).
6. **Merge:** Dev aprova, CI atualiza tag da imagem no repo de manifestos (`app-config`).
7. **ArgoCD:** Detecta mudança, sincroniza manifestos (K8s pull model).
8. **Argo Rollouts:** Executa Canary (10% → 5min pausa → 50% → 100%).
9. **Prometheus:** Monitora erro%, latência p99. Se erro > 1% ou latência p99 > 500ms → AlertManager → rollback automático.
10. **Grafana:** Dashboard mostra mudança em real-time (DORA metrics, Golden Signals, business KPIs).

---

## Decisões Tecnológicas & Justificativas

| Escolha | Alternativa Rejeitada | Razão |
|---------|----------------------|-------|
| **Go** | Python FastAPI | Latência crítica para notificações; binários estáticos; menor overhead de recursos em containers; startup < 100ms. |
| **Terraform** | CloudFormation / Pulumi | Linguagem agnóstica (não vendor-lock AWS); comunidade maior; HCL legível; melhor para módulos reutilizáveis. |
| **K3s (dev)** | Kind / Minikube | K3s é production-ready mesmo localmente; melhor performance; single binary; redis+etcd built-in. |
| **ArgoCD** | Flux / Helm | GitOps native; UI web intuitiva; ApplicationSet para multi-env; rollback visual; melhor observabilidade. |
| **Prometheus** | Datadog / New Relic | Open-source, integrado com K8s nativamente; CNCF; custo previsível; Grafana pode escalar indefinidamente. |
| **Loki** | ELK / Splunk | Horizontal scaling eficiente; compatible com Prometheus labels; custo baixo em volume alto; perfil de memória pequeno. |
| **Argo Rollouts** | Service Mesh (Istio) | Sem vendor-lock de mesh; análise integrada; suporta Canary/Blue-Green/Progressive sem complexidade extra. |
| **OPA/Gatekeeper** | Kyverno | OPA é padrão CNCF; mais flexível; policy-as-code em Rego; integra com ArgoCD em tempo de sync. |

## Instruções para Colaboração
Ao me auxiliar neste projeto:
1. **Priorize Modularidade:** Terraform modules, K8s Helm charts, Go packages bem delimitados.
2. **Foco em Segurança:** Non-root, RBAC, Pod Security Policy, NetworkPolicy, secrets encrypted at rest.
3. **DevEx:** Se manual, automatize via Taskfile/script/GitHub Actions. Tempo é custo.
4. **Custo:** Sempre mencione impacto $$. Favor LocalStack/K3s em dev. Tag recursos com `cost-center`.
5. **Observabilidade:** Toda feature deve ter métrica, log e trace. SLO deve guiar deploy.
6. **Go Style:** Efetivo Go, interfaces pequenas, error handling explícito, sem magic.
```

---

### Como usar este arquivo:
1. Salve o conteúdo acima como `context.md` na raiz do seu repositório.
2. Sempre que iniciar um novo chat com uma LLM, **anexe ou cole** este arquivo e diga: 
   > *"Baseie-se neste contexto para me ajudar nas próximas tarefas deste projeto."*