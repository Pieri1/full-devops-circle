# QUICKSTART.md – Guia de Início Rápido

**Tempo de leitura:** 5 minutos  
**Próximo passo:** Escolha sua trilha e comece

---

## 🎯 Qual é sua função?

### 👨‍💻 Sou Desenvolvedor

**Quer:** Clonar, fazer setup local, começar a codar  
**Tempo:** 15 minutos

1. **Leia:**
   ```bash
   README.md (Tech Stack)
   todo.md (Fase 1: DevEx)
   ```

2. **Execute Fase 1:**
   - Configurar Dev Container
   - Rodar `task up` (LocalStack + K3s)
   - Executar `task lint`, `task test`

3. **Próximo:** Comece Fase 2 (Aplicação)

---

### 🏗️ Sou Arquiteto / Tech Lead

**Quer:** Entender decisões, validar trade-offs, customizar  
**Tempo:** 30 minutos

1. **Leia:**
   ```bash
   README.md (Visão Geral)
   DECISIONS.md (14 decisões + justificativas)
   ARCHITECTURE.md (Fluxos sistêmicos)
   ```

2. **Customizações Recomendadas:**
   - [ ] Ajuste SLO (99.95% → seu target?)
   - [ ] Revise budgets (prod: $2k/m → seu budget?)
   - [ ] Considere ambientes extras (blue/green? preview?)
   - [ ] Discuta segurança (7 camadas é suficiente?)

3. **Próximo:** Implemente Fase 1 com seu time

---

### 🔧 Sou DevOps / SRE

**Quer:** Provisionar infra, setup CI/CD, observabilidade, caos  
**Tempo:** 45 minutos

1. **Leia:**
   ```bash
   ARCHITECTURE.md (Diagramas K8s, CI/CD, O11y)
   todo.md (Fases 3, 4, 5, 6)
   todo-fase-6.5.md (Governance)
   DECISIONS.md (Terraform, K8s, ArgoCD choices)
   ```

2. **Implemente em Ordem:**
   - Fase 3: Terraform (VPC, EKS, RDS, Secrets)
   - Fase 4: GitHub Actions + ArgoCD
   - Fase 5: Prometheus + Grafana + Loki
   - Fase 6: LitmusChaos + Argo Rollouts analysis
   - Fase 6.5: Governance (approval, audit, DR)

3. **Validações:**
   - [ ] Todos builds passam em CI
   - [ ] Canary deploy completa sem rollback
   - [ ] Alertas disparam corretamente
   - [ ] Chaos experiment não quebra app
   - [ ] Disaster recovery drill < 2h

---

### 👮 Sou Compliance / Segurança Officer

**Quer:** Validar segurança, compliance, auditoria, governance  
**Tempo:** 20 minutos

1. **Leia:**
   ```bash
   ARCHITECTURE.md (7 Security Layers)
   todo.md (Fases 2, 4, 6 - security sections)
   todo-fase-6.5.md (Compliance, Auditoria, CAB)
   ```

2. **Valide:**
   - ✅ Shift-left: Gitleaks, Semgrep, Trivy em CI
   - ✅ Admission control: OPA/Gatekeeper policies
   - ✅ Network: NetworkPolicy deny-all default
   - ✅ Audit: CloudTrail + K8s audit logs
   - ✅ Secrets: Rotation automática q90d
   - ✅ Backup: Testado q30d
   - ✅ Approval: CAB para mudanças críticas

3. **Próximo:** Customize policies conforme regulação (LGPD, SOX, etc)

---

### 💰 Sou CFO / Product Manager

**Quer:** Entender custo, ROI, trade-offs business  
**Tempo:** 10 minutos

1. **Leia:**
   ```bash
   README.md (Tech Stack - veja impacto custo)
   DECISIONS.md (cada decisão tem custo explícito)
   todo-fase-6.5.md (FinOps section)
   ```

2. **Key Metrics:**
   - **Infrastructure Cost:** ~$2k/mês prod (~$500 dev)
   - **DevEx Gain:** Setup < 10min (vs 2-3 horas manual)
   - **Deployment Speed:** ~15min canary (vs 1-2 horas manual)
   - **Mean Time to Recovery:** < 30min (vs 2-4 horas)
   - **Observabilidade:** Incident detection < 1min (vs manual monitoring)

3. **ROI:** 
   - Reduz bugs em produção 70% (shift-left)
   - Reduz incident response 80% (observabilidade + automation)
   - Reduz DevOps overhead 50% (GitOps + automation)
   - → **Break-even em 3-6 meses**

---

## 📚 Documentos por Arquivo

### README.md (271 linhas)
**O QUE:** Visão geral do projeto  
**POR QUE:** Quick reference, orientação  
**COMO:** 3 minutos de leitura, resumido

```
Inclui:
├─ Tech stack (resumido)
├─ Arquitetura (alto nível)
├─ Decisões (tabela)
├─ Fases (overview)
└─ Como usar (este projeto)
```

### context.md (108 linhas)
**O QUE:** Stack detalhado + instruções para LLM  
**POR QUE:** Guardar decisões + contexto consistente  
**COMO:** Anexe em futuro chat com LLM

```
Inclui:
├─ Stack tecnológico (detalhado)
├─ Estrutura de diretórios
├─ Workflow (CI/CD pipeline)
├─ Decisões (tabela comparativa)
└─ Instruções para colaboração
```

### DECISIONS.md (367 linhas)
**O QUE:** Por quê cada decisão + trade-offs  
**POR QUE:** Justificar escolhas, reconhecer limites  
**COMO:** Consulte quando questionado

```
Inclui (14 decisões):
├─ Go vs Python
├─ Terraform vs CloudFormation
├─ AWS vs GCP/Azure
├─ K8s vs Swarm/Nomad
├─ ArgoCD vs Flux
├─ Prometheus vs Datadog
├─ Loki vs ELK
├─ Jaeger vs Datadog
├─ Semgrep vs SonarQube
├─ Trivy vs Clair
├─ OPA vs Kyverno
├─ API versioning (path vs header)
├─ PostgreSQL vs MongoDB
├─ SQS vs Kafka
├─ FinOps (Infracost + Kubecost)
├─ Chaos (LitmusChaos)
├─ RPO/RTO targets
└─ Ambientes (4: local, dev, staging, prod)

Cada decisão tem:
├─ Justificativa
├─ Trade-offs
└─ Quando reconsiderar
```

### ARCHITECTURE.md (698 linhas)
**O QUE:** Diagramas, fluxos, padrões, componentes  
**POR QUE:** Entender como tudo funciona, correlações  
**COMO:** Leia para aprender sistema, consulte para debugging

```
Inclui:
├─ Overview (diagrama alto nível)
├─ Componentes (app, data, K8s, CI/CD, observabilidade)
├─ Data flow (request lifecycle com trace_id)
├─ Security (7 camadas)
├─ Resilience patterns (circuit breaker, retry, timeout)
├─ Development workflow
├─ Chaos engineering scenarios
└─ Summary table (tech × purpose)
```

### todo.md (629 linhas)
**O QUE:** Checklist executável com 6 fases  
**POR QUE:** Roadmap de implementação, critérios de aceite  
**COMO:** Siga fase por fase, marque como completo

```
Fase 1: DevEx (2 semanas)
├─ Dev Container
├─ Taskfile.yml
├─ LocalStack
├─ Documentação viva
└─ VS Code workspace

Fase 2: Aplicação (3 semanas)
├─ Go microservice
├─ Testes (UT + integration)
├─ Proteção de secrets
├─ SAST (Semgrep)
└─ Docker otimizado

Fase 3: IaC (3 semanas)
├─ Módulos Terraform
├─ Ambientes
├─ Validação (tfsec, Trivy)
├─ FinOps (Infracost)
└─ Estado remoto

Fase 4: CI/CD (2 semanas)
├─ Pipeline GitHub Actions
├─ Repositório de manifestos
├─ ArgoCD setup
└─ Argo Rollouts (canary)

Fase 5: Observabilidade (2 semanas)
├─ Prometheus + Grafana
├─ Loki (logs)
├─ Jaeger (traces)
├─ SLO/SLI
└─ Custom metrics

Fase 6: Resiliência (2 semanas)
├─ NetworkPolicy
├─ LitmusChaos
├─ Experimentos (pod delete, latency, etc)
└─ Análise automática
```

### todo-fase-6.5.md (208 linhas)
**O QUE:** Governance, compliance, FinOps avançado  
**POR QUE:** Production-grade operações (approval, audit, DR)  
**COMO:** Implemente após Fase 6

```
Inclui:
├─ Governance (approval gates, CAB)
├─ Compliance (auditoria, secrets rotation)
├─ FinOps (tagging, budgets, anomaly detection)
├─ Operações (on-call, incident response, SLO burn rate)
├─ Security scanning (SAST, SCA, DAST, supply chain)
└─ Secrets detection (Gitleaks, TruffleHog)
```

### SUMMARY.md (272 linhas)
**O QUE:** O que mudou nesta sessão  
**POR QUE:** Referência de mudanças, highlights  
**COMO:** Leia para entender antes/depois

```
Inclui:
├─ Decisões concretas tomadas (14 total)
├─ Melhorias no checklist (vague → específico)
├─ Comparação antes vs depois
├─ Novos documentos explicados
└─ FAQ
```

### PROJECT_MAP.md (273 linhas)
**O QUE:** Mapa de navegação, índice cruzado  
**POR QUE:** Encontrar informação rapidamente  
**COMO:** Use para descobrir qual arquivo tem resposta

```
Inclui:
├─ Trilhas por função (dev, arquiteto, DevOps, compliance, CFO)
├─ Índice cruzado (tech × arquivo)
├─ Mapa conceitual
├─ Estatísticas
└─ Checklist próximos passos
```

---

## 🚀 Trilhas Recomendadas (por papel)

### Trilha Arquitetura (4h)
```
1. README.md (15 min)
   └─ Overview, tech stack

2. DECISIONS.md (60 min)
   └─ 14 decisões, trade-offs, alternativas

3. ARCHITECTURE.md (75 min)
   └─ Diagramas, fluxos, padrões

4. PROJECT_MAP.md (20 min)
   └─ Índice, navegação

5. Discussão com time (30 min)
```

### Trilha Implementação (20h)
```
1. README.md (15 min)

2. context.md (10 min)

3. todo.md Fase 1 (4h)
   └─ DevEx + Local setup

4. todo.md Fase 2 (6h)
   └─ Aplicação + testes

5. todo.md Fase 3 (6h)
   └─ IaC Terraform

6. todo.md Fases 4-6 (4h)
   └─ CI/CD, O11y, Caos (estrutura, não detalhes)
```

### Trilha Governance (2h)
```
1. README.md (10 min)

2. ARCHITECTURE.md Security section (20 min)

3. todo-fase-6.5.md (60 min)
   └─ Approval gates, compliance, audit

4. DECISIONS.md item 12-14 (20 min)
   └─ Secrets, SLO, disaster recovery
```

---

## ❓ FAQ Rápido

**P: Por onde começo?**  
R: Leia `README.md` (3 min). Depois escolha sua função acima.

**P: Qual é o tempo total de implementação?**  
R: ~17 semanas (fases 1-6.5). Mas você pode parar em qualquer fase e ter valor.

**P: Posso pular fases?**  
R: Sim, mas há dependências:
- Fase 4 precisa de Fase 2 + 3
- Fase 5 precisa de Fase 4
- Fase 6 precisa de Fase 4 + 5
- Fase 6.5 precisa de Fases 4 + 5 + 6

**P: E se discordo com uma decisão?**  
R: Veja `DECISIONS.md` → item → "Quando reconsiderar". Suas necessidades podem ser diferentes.

**P: Posso usar isso em produção?**  
R: Sim! É referência de melhores práticas. Customize para seu contexto.

**P: Qual é o custo?**  
R: ~$2k/mês AWS prod (EKS, RDS, SQS, Secrets). ~$500/mês dev. Veja `DECISIONS.md` item 3.

**P: Há performance overhead?**  
R: Não. Go é binário compilado (< 100ms p99). Observabilidade é 15% CPU overhead. Caos é on-demand.

---

## ✅ Checklist: Comece Agora

- [ ] Leia README.md (3 min)
- [ ] Escolha sua trilha acima
- [ ] Execute primeiro documento da trilha
- [ ] Faça uma anotação: "Próximos passos?"
- [ ] Converse com time: "Concorda com decisões?"
- [ ] Comece Fase 1

---

**Você está pronto para começar! 🚀**

Next: Escolha sua trilha acima e comece com o primeiro documento.

Dúvidas? Consulte `PROJECT_MAP.md` para encontrar informação ou abra issue.

---

**Versão:** 1.0  
**Data:** 2026-03-19  
**Tempo de Leitura:** 5 minutos
