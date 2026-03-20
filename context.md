### `context.md`

```markdown
# Contexto do Projeto: Ciclo Completo DevOps & DevEx

## Objetivo do Projeto
Construir uma infraestrutura robusta e automatizada para um microserviço de **Notificações**, focando em Developer Experience (DevEx), automação de infraestrutura (IaC), entrega contínua (GitOps) e observabilidade avançada. O projeto visa simular um ambiente de produção real com resiliência e segurança.

---

## Stack Tecnológica

### 1. Aplicação & DevEx
- **Linguagem:** Go (ou Python FastAPI) para alta performance.
- **Ambiente Local:** Dev Containers (Docker) para isolamento total do setup.
- **Automação de CLI:** `Taskfile` ou `Makefile` para padronização de comandos.
- **Documentação:** Swagger/OpenAPI + Mermaid.js para diagramas de arquitetura.

### 2. Infraestrutura como Código (IaC) & Cloud
- **Provisionamento:** Terraform / OpenTofu.
- **Ambiente de Nuvem:** AWS (ou LocalStack para simulação local de S3/SQS/RDS).
- **Orquestração:** Kubernetes (K3s ou Kind para desenvolvimento).

### 3. CI/CD & GitOps
- **Pipeline de CI:** GitHub Actions (Lint, Testes unitários com Testcontainers, Scan de Vulnerabilidades).
- **Entrega (CD):** ArgoCD seguindo o modelo GitOps.
- **Estratégia de Deploy:** Progressive Delivery (Canary) via Argo Rollouts.

### 4. Segurança (DevSecOps)
- **SAST/SCA:** Trivy e Semgrep integrados ao pipeline.
- **Secrets:** Gitleaks para prevenir vazamento de chaves.
- **FinOps:** Infracost para estimativa de custos em Pull Requests.

### 5. Observabilidade & Resiliência
- **Métricas/Dashboards:** Prometheus + Grafana (Foco em DORA Metrics e Golden Signals).
- **Logs:** Grafana Loki.
- **Tracing:** OpenTelemetry.
- **Engenharia de Caos:** LitmusChaos para teste de resiliência.

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
1. O desenvolvedor altera o código no **Dev Container**.
2. O **CI** valida código, segurança e gera uma imagem Docker.
3. O **Infracost** analisa mudanças no Terraform e reporta o custo no PR.
4. O **ArgoCD** sincroniza o estado desejado no Cluster K8s.
5. O **Argo Rollouts** executa um Canary Deploy.
6. Se o **Prometheus** detectar erro > 1%, o **Rollback** é automático.
7. O **Grafana** exibe o impacto da mudança em tempo real.

---

## Instruções para a LLM
Ao me auxiliar neste projeto:
1. **Priorize Modularidade:** No Terraform e no Kubernetes, utilize módulos e separação de interesses.
2. **Foco em Segurança:** Sempre sugira boas práticas (non-root containers, RBAC, etc.).
3. **Considere o Custo:** Ao sugerir recursos de nuvem, mencione o impacto financeiro ou alternativas no LocalStack.
4. **DevEx em Primeiro Lugar:** Se um processo for manual, sugira como automatizá-lo via Taskfile ou script.
5. **Estilo de Código:** Siga os padrões idiomáticos da linguagem escolhida (ex: Go standard layout).
```

---

### Como usar este arquivo:
1. Salve o conteúdo acima como `context.md` na raiz do seu repositório.
2. Sempre que iniciar um novo chat com uma LLM, **anexe ou cole** este arquivo e diga: 
   > *"Baseie-se neste contexto para me ajudar nas próximas tarefas deste projeto."*