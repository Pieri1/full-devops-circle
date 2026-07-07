# PROJECT_MAP.md – Mapa Mental do Projeto

Guia visual para navegar toda documentação e entender como tudo se conecta.

---

## 📍 Estrutura de Arquivos

```
/home/pieri/projects/full-devops-circle/
│
├── README.md                 ← START HERE (visão geral, stack, quick reference)
│                              └─ Contém: Objetivo, tech stack resumido, fases, decisões
│
├── context.md                ← REFERÊNCIA (stack detalhado, instruções para LLM)
│                              └─ Contém: Stack tecnológico, workflow, decisões de design
│
├── DECISIONS.md              ← JUSTIFICATIVAS (por quê cada tecnologia)
│                              └─ Contém: 14 decisões, trade-offs, quando reconsiderar
│
├── ARCHITECTURE.md           ← FLUXOS & DIAGRAMAS (como tudo funciona)
│                              └─ Contém: Componentes, request lifecycle, segurança, caos
│
├── todo.md                   ← CHECKLIST EXECUTÁVEL (fases 1-6)
│                              └─ Contém: 6 fases com sub-tarefas + critérios de aceitação
│
├── todo-fase-6.5.md          ← GOVERNANCE & COMPLIANCE
│                              └─ Contém: Approval gates, auditoria, disaster recovery, FinOps
│
└── SUMMARY.md                ← MUDANÇAS & RESUMO DESTA SESSÃO
                               └─ Contém: O que foi feito, decisões, comparação antes/depois
```

---

## 🗺️ Como Navegar

### Você é...

#### **Novo no Projeto?**
```
README.md (3 min)
    ↓
context.md (5 min)
    ↓
ARCHITECTURE.md (10 min) – Diagramas
    ↓
Comece Fase 1 do todo.md
```

#### **Arquiteto/Tech Lead?**
```
DECISIONS.md (15 min) – Entender trade-offs
    ↓
ARCHITECTURE.md (20 min) – Fluxos sistêmicos
    ↓
context.md (5 min) – Instruções para LLM
    ↓
Customize: SLO, budgets, ambientes
```

#### **DevOps/SRE?**
```
todo.md (30 min) – Fases 3, 4, 5, 6
    ↓
todo-fase-6.5.md (15 min) – Governance
    ↓
ARCHITECTURE.md (10 min) – Observabilidade, segurança
    ↓
Implemente as fases em ordem
```

#### **Desenvolvedor?**
```
README.md (3 min)
    ↓
todo.md Fase 1 (10 min) – DevEx
    ↓
ARCHITECTURE.md (5 min) – Request lifecycle, observabilidade
    ↓
Comece a codar em Fase 2
```

#### **Quer Justificar uma Decisão?**
```
DECISIONS.md → Busque a decisão
    ↓ Se questionado:
    ├─ Justificativa (por quê escolhemos)
    ├─ Trade-offs (o que perdemos)
    └─ Quando reconsiderar (reconhecendo limites)
```

#### **Quer Entender um Componente (ex: ArgoCD)?**
```
ARCHITECTURE.md → Busque "GitOps Deployment Flow"
    ↓
DECISIONS.md → Busque "4. CI/CD: GitHub Actions + ArgoCD..."
    ↓
context.md → Veja como integra no workflow
    ↓
todo.md Fase 4 → Implemente
```

---

## 🎯 Mapa de Conteúdo

```
┌─────────────────────────────────────────────────────────────────┐
│                      PROJETO: Notificações                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐  │
│ │   README.md      │  │  context.md      │  │  DECISIONS.md  │  │
│ │  ────────────── │  │  ──────────────  │  │  ────────────  │  │
│ │ • Visão geral    │  │ • Stack detalh.  │  │ • 14 decisões  │  │
│ │ • Tech stack     │  │ • Workflow       │  │ • Trade-offs   │  │
│ │ • Fases (1-6.5)  │  │ • Instruções LLM │  │ • Alternativas │  │
│ │ • Quick ref      │  │ • Tabela decisões│  │ • When rethink │  │
│ └──────────┬───────┘  └────────┬─────────┘  └────────┬───────┘  │
│            │                   │                     │           │
│            └───────────────────┼─────────────────────┘           │
│                                ↓                                 │
│                        ┌───────────────────┐                     │
│                        │  ARCHITECTURE.md  │                     │
│                        │  ───────────────  │                     │
│                        │ • 6 diagramas     │                     │
│                        │ • App layer       │                     │
│                        │ • Data layer      │                     │
│                        │ • K8s             │                     │
│                        │ • CI/CD flow      │                     │
│                        │ • Observabilidade │                     │
│                        │ • Security (7x)   │                     │
│                        │ • Resilience      │                     │
│                        │ • Chaos scenarios │                     │
│                        └────────┬──────────┘                     │
│                                 │                                │
│              ┌──────────────────┼──────────────────┐             │
│              ↓                  ↓                  ↓             │
│      ┌─────────────┐   ┌──────────────┐  ┌────────────────┐    │
│      │  todo.md    │   │todo-fase-6.5 │  │  SUMMARY.md    │    │
│      │ ─────────── │   │  ───────────  │  │  ────────────  │    │
│      │ Fase 1: DevEx   │ • Governance  │  │• O que mudou   │    │
│      │ Fase 2: App    │ • Compliance  │  │• Decisões      │    │
│      │ Fase 3: IaC    │ • Auditoria   │  │• Comparação    │    │
│      │ Fase 4: CI/CD  │ • FinOps      │  │• Highlights    │    │
│      │ Fase 5: O11y   │ • On-call     │  │• FAQ           │    │
│      │ Fase 6: Chaos  │ • DR drills   │  │                │    │
│      │ Checklist      │ • Success     │  │                │    │
│      │ Executável     │   criteria    │  │                │    │
│      └────────────────┘  └──────────────┘  └────────────────┘    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔍 Índice Cruzado

### Tech Stack

| Tecnologia | Mencionada em | Justificativa em | Implementação em |
|-----------|--------------|-----------------|-----------------|
| **Go 1.23** | README, context | DECISIONS (item 1) | todo.md fase 2 |
| **Terraform** | README, context | DECISIONS (item 2) | todo.md fase 3 |
| **AWS** | context | DECISIONS (item 3) | todo.md fase 3 |
| **Kubernetes** | README, ARCH | DECISIONS (item 4) | todo.md fases 1,3,4 |
| **ArgoCD** | README, ARCH | DECISIONS (item 5) | todo.md fase 4 |
| **Argo Rollouts** | README, ARCH | DECISIONS (item 5) | todo.md fases 4,6 |
| **Prometheus** | README, ARCH | DECISIONS (item 6) | todo.md fase 5 |
| **Grafana** | README, ARCH | DECISIONS (item 6) | todo.md fase 5 |
| **Loki** | README, ARCH | DECISIONS (item 6) | todo.md fase 5 |
| **Semgrep** | README, ARCH | DECISIONS (item 7) | todo.md fase 2 |
| **Trivy** | README, ARCH | DECISIONS (item 7) | todo.md fases 2,4 |
| **Gitleaks** | README, ARCH | DECISIONS (item 7) | todo.md fase 2 |
| **LitmusChaos** | README, ARCH | DECISIONS (item 10) | todo.md fase 6 |
| **PostgreSQL** | context | DECISIONS (item 9) | todo.md fases 2,3 |
| **SQS** | context | DECISIONS (item 11) | todo.md fases 2,3 |
| **Secrets Manager** | context | DECISIONS (item 12) | todo.md fases 3,4 |
| **Infracost** | README, ARCH | DECISIONS (item 8) | todo.md fases 3,4 |
| **Kubecost** | README, ARCH | DECISIONS (item 8) | todo-fase-6.5 |

### Conceitos-Chave

| Conceito | Explicado em | Implementado em |
|---------|------------|-----------------|
| **GitOps Pull Model** | ARCHITECTURE (GitOps flow) | todo.md fase 4 |
| **Canary Deploy** | ARCHITECTURE (deployment seq) | todo.md fase 4 + ARCHITECTURE (chaos) |
| **SLO/SLI** | ARCHITECTURE (SLO tracking) | todo.md fase 5 + todo-fase-6.5 |
| **Golden Signals** | ARCHITECTURE (o11y stack) | todo.md fase 5 |
| **Circuit Breaker** | ARCHITECTURE (resilience) | todo.md fase 2 + fase 6 |
| **Network Policy** | ARCHITECTURE (security) | todo.md fase 6 |
| **Shift-Left Security** | DECISIONS (item 7) | todo.md fase 2 + 4 |
| **7 Security Layers** | ARCHITECTURE (security layers) | todo.md fases 2,3,4,6,6.5 |

### Fases

| Fase | Arquivo | Duração Est. | Dependencies |
|------|---------|-------------|--------------|
| **1: DevEx** | todo.md | 2 semanas | Nenhuma |
| **2: App** | todo.md | 3 semanas | Fase 1 |
| **3: IaC** | todo.md | 3 semanas | Fase 1 |
| **4: CI/CD** | todo.md | 2 semanas | Fases 2,3 |
| **5: O11y** | todo.md | 2 semanas | Fase 4 |
| **6: Chaos** | todo.md | 2 semanas | Fases 4,5 |
| **6.5: Governance** | todo-fase-6.5 | 3 semanas | Fases 4,5,6 |
| **Total** | - | ~17 semanas | - |

---

## 📊 Estatísticas

```
Documentação Gerada:
├─ Total de Linhas: 2,553
├─ Total de Arquivos: 7
├─ Tamanho Total: ~130 KB
│
Breakdown:
├─ ARCHITECTURE.md ........ 698 linhas (diagramas, fluxos)
├─ todo.md ............... 629 linhas (fases 1-6, checklist)
├─ DECISIONS.md .......... 367 linhas (14 decisões + trade-offs)
├─ SUMMARY.md ............ 272 linhas (mudanças desta sessão)
├─ README.md ............. 271 linhas (visão geral)
├─ todo-fase-6.5.md ...... 208 linhas (governance)
└─ context.md ............ 108 linhas (updated)

Cobertura:
├─ Stack Tecnológico: ✅ 14 decisões documentadas
├─ Arquitetura: ✅ 6 diagramas + fluxos
├─ Implementação: ✅ 6.5 fases com critérios
├─ Governance: ✅ 13 sub-tarefas fase 6.5
├─ Segurança: ✅ 7 camadas detalhadas
└─ Observabilidade: ✅ 4 pilares (métrica, log, trace, alert)
```

---

## 🚀 Checklist: Próximos Passos

- [ ] Leia README.md (3 min)
- [ ] Consulte DECISIONS.md para questões de design (15 min)
- [ ] Estude ARCHITECTURE.md para entender fluxos (20 min)
- [ ] Customize SLOs, budgets, ambientes conforme seu contexto
- [ ] Implemente Fase 1 (DevEx) do todo.md
- [ ] Revise Fase 2-6 com seu time
- [ ] Discuta trade-offs com stakeholders
- [ ] Comece implementação após alignment

---

## 💬 Comunicação com LLM

Copie este arquivo para futuras sesões:

```markdown
Estou trabalhando no projeto "full-devops-circle".

Contexto fornecido em:
- `/home/pieri/projects/full-devops-circle/context.md`
- `/home/pieri/projects/full-devops-circle/ARCHITECTURE.md`
- `/home/pieri/projects/full-devops-circle/DECISIONS.md`
- `/home/pieri/projects/full-devops-circle/todo.md`
- `/home/pieri/projects/full-devops-circle/todo-fase-6.5.md`

Use esses documentos como referência para me ajudar com...
```

---

**Versão:** 1.0  
**Data:** 2026-03-19  
**Status:** ✅ Mapa Completo e Navegável
