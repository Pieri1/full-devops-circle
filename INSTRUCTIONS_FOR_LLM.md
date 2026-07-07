# Para Próximas Sessões com LLM

Se você quer continuar trabalhando neste projeto em futuras conversas com IA, copie e cole o texto abaixo no chat:

---

## Contexto do Projeto

Estou trabalhando no projeto **full-devops-circle**: um microserviço de notificações com stack completo de DevOps, DevEx, GitOps, Observabilidade e Engenharia de Caos.

**Localização:** `/home/pieri/projects/full-devops-circle/`

### Documentação Principal

Para auxílio técnico, use como referência:

1. **context.md** – Stack tecnológico decidido, workflow, instruções
2. **DECISIONS.md** – 14 decisões de design com trade-offs explícitos
3. **ARCHITECTURE.md** – Diagramas, fluxos, componentes, padrões
4. **todo.md** – Fases 1-6 com checklist executável
5. **todo-fase-6.5.md** – Governance, compliance, FinOps, disaster recovery

### Stack Tecnológico (DECISÕES VINCULANTES)

- **Linguagem:** Go 1.23 (binários estáticos, latência < 100ms)
- **IaC:** Terraform 1.6+ (módulos reutilizáveis)
- **Nuvem:** AWS (prod) + LocalStack (dev)
- **Orquestração:** Kubernetes (EKS prod, K3s dev)
- **GitOps:** ArgoCD + Argo Rollouts (canary automático)
- **Observabilidade:** Prometheus + Grafana + Loki + Jaeger
- **Segurança:** 7 camadas (shift-left + admission control + audit)
- **Resiliência:** LitmusChaos + auto-rollback Argo Rollouts
- **FinOps:** Infracost + Kubecost + tagging obrigatório
- **SLO:** 99.95% availability, p99 latência < 200ms

### Estrutura do Projeto

```
├── context.md              # Stack + instruções
├── DECISIONS.md            # Decisões com trade-offs
├── ARCHITECTURE.md         # Diagramas + fluxos
├── todo.md                 # Fases 1-6 (checklist)
├── todo-fase-6.5.md        # Governance
├── README.md               # Visão geral
├── QUICKSTART.md           # Guia por função
├── PROJECT_MAP.md          # Navegação
└── SUMMARY.md              # Mudanças v1.0
```

### Como Usar Este Contexto

- **Para mudanças de design:** Consulte `DECISIONS.md` (trade-offs + alternativas)
- **Para entender sistema:** Leia `ARCHITECTURE.md` (diagramas + fluxos)
- **Para implementar:** Siga `todo.md` (fases em ordem)
- **Para governance:** Consulte `todo-fase-6.5.md` (compliance, audit, DR)
- **Para navegar:** Use `PROJECT_MAP.md` (índice cruzado) ou `QUICKSTART.md` (trilhas)

---

## Instruções para Colaboração

Ao me pedir ajuda, seja específico:

### Exemplos Bons

```
"Estou implementando Fase 3 (IaC). Como estruturar módulos Terraform 
para que sejam reutilizáveis entre dev/staging/prod? 
Veja referência em todo.md seção 3.1."

"A decisão de usar Argo Rollouts tem alguma limitação conhecida?
Consulte DECISIONS.md item 4 para trade-offs."

"Qual é o request lifecycle completo da API? Preciso instrumentar
com OpenTelemetry. Veja ARCHITECTURE.md seção 'Data Flow'."
```

### Exemplos Ruins

```
"Help me with DevOps"  (vague, sem contexto)
"Do CI/CD setup"  (qual é a abordagem? Veja DECISIONS.md)
"Make everything observable"  (como? Veja ARCHITECTURE.md + todo.md fase 5)
```

### Formato de Requisição Ideal

```
[CONTEXTO] Estou na Fase X (do projeto full-devops-circle)
[PROBLEMA] Preciso fazer Y
[REFERÊNCIA] Veja Z em [arquivo].md seção [seção]
[ESPECÍFICO] Minha dúvida é...
```

---

## Tópicos Comuns

### Se Questionarem uma Decisão

→ Consulte `DECISIONS.md`  
→ Seção "Trade-offs" + "Quando reconsiderar"  
→ Exemplo: "Por Go? Veja item 1 em DECISIONS.md"

### Se Precisar de Arquitetura

→ Consulte `ARCHITECTURE.md`  
→ Encontre diagrama relevante  
→ Exemplo: "GitOps flow? Veja ARCHITECTURE.md seção 'GitOps Deployment Flow'"

### Se Quiser Implementar uma Fase

→ Consulte `todo.md`  
→ Fase relevante + sub-tarefas + critérios  
→ Exemplo: "Implemente Fase 4? Siga Fase 4 em todo.md passo a passo"

### Se Precisar de Governance/Compliance

→ Consulte `todo-fase-6.5.md`  
→ Seção relevante (governance, compliance, auditoria, etc)  
→ Exemplo: "Como estruturar approval gates? Veja Fase 6.5 seção '6.5.1'"

### Se Estiver Perdido

→ Consulte `QUICKSTART.md` (trilhas por função)  
→ Ou `PROJECT_MAP.md` (índice cruzado)  
→ Exemplo: "Sou DevOps, por onde começo? Veja QUICKSTART.md trilha DevOps"

---

## Versão e Histórico

**Versão Atual:** 1.0 (2026-03-19)

### Mudanças da v1.0

- ✅ Decisões concretas (não vague): Go, Terraform, AWS + LocalStack, etc
- ✅ Trade-offs explícitos: Cada decisão reconhece limites
- ✅ Critérios mensuráveis: SLO 99.95%, p99 < 200ms, etc
- ✅ 7 camadas de segurança: Shift-left + admission + audit
- ✅ Fase 6.5 completa: Governance, compliance, disaster recovery
- ✅ Documentação navegável: PROJECT_MAP.md + QUICKSTART.md
- ✅ 9 arquivos, 3.099 linhas, ~150 KB

### Se Houver Mudanças Futuras

- Atualize `context.md` se stack muda
- Atualize `DECISIONS.md` se reconsiderar escolha
- Atualize `todo.md` se critérios mudam
- Mantenha `ARCHITECTURE.md` sincronizado com implementação
- Registre em novo `SUMMARY-v2.md`

---

## Contato / Escalação

Se encontrar:

- **Gap em documentação:** Abra issue ou crie novo `.md`
- **Dúvida não respondida:** Consulte PROJECT_MAP.md (índice)
- **Conflito de design:** Veja DECISIONS.md (trade-offs)
- **Problema implementação:** Veja ARCHITECTURE.md + todo.md (sequência)

---

## Quick Links

| Preciso de... | Consulte |
|-------------|----------|
| Visão geral | README.md |
| Por onde começo? | QUICKSTART.md |
| Encontrar informação | PROJECT_MAP.md |
| Stack detalhado | context.md |
| Justificativas | DECISIONS.md |
| Diagramas + fluxos | ARCHITECTURE.md |
| Implementar fase | todo.md |
| Governance/compliance | todo-fase-6.5.md |
| Mudanças v1.0 | SUMMARY.md |

---

**Copie este arquivo para próximas sessões! Ele facilita muito a colaboração.** 🚀

