# Checklist Avançado: Projeto DevOps Full-Cycle

Este documento define as etapas, ferramentas e critérios de aceitação para o desenvolvimento de um ambiente de microserviços de alto padrão, englobando DevEx, Segurança, IaC, GitOps, Observabilidade e Resiliência.

---

## Fase 1: Developer Experience (DevEx) e Fundação
O objetivo desta fase é garantir que qualquer pessoa clone o repositório e comece a produzir em menos de 5 minutos, com um ambiente totalmente declarativo e documentado.

- [ ] **1. Ambiente Reproduzível:** Configurar o ambiente de desenvolvimento local de forma declarativa (ex: utilizando um `flake.nix` ou `shell.nix` para garantir que dependências como Go/Rust, Terraform, K3s, e CLI tools estejam disponíveis na versão exata, sem poluir o sistema host).
- [ ] **2. Workspace Integrado:** Configurar o workspace do Visual Studio Code (`.vscode/settings.json` e `extensions.json`) com linters, formatadores e integração com GitHub Copilot ativados por padrão.
- [ ] **3. Documentação Viva:** Desenhar a arquitetura do sistema utilizando `Mermaid.js` diretamente no `README.md`.
- [ ] **4. Automação de Tarefas:** Criar um `Taskfile.yml` com comandos padronizados (`task up`, `task lint`, `task test`).
- [ ] **5. Mock de Nuvem Local:** Configurar o LocalStack via Docker/Podman para simular serviços gerenciados (S3, SQS, etc.) localmente.

---

## Fase 2: Aplicação e Shift-Left Security (DevSecOps)
A aplicação deve ser construída pensando em performance e segurança desde o primeiro commit.

- [ ] **1. Microserviço Base:** Desenvolver uma API REST/gRPC simples (ex: Serviço de Notificações) com health checks (`/health/liveness` e `/health/readiness`).
- [ ] **2. Testes Resilientes:** Implementar testes de integração isolados utilizando **Testcontainers** (subindo o banco de dados temporariamente no Docker durante o teste).
- [ ] **3. Proteção de Segredos:** Instalar o **Gitleaks** via `pre-commit hook` para bloquear commits contendo chaves ou senhas em texto plano.
- [ ] **4. Análise Estática (SAST):** Configurar o **Semgrep** para rodar localmente e barrar anti-patterns ou vulnerabilidades lógicas.
- [ ] **5. Containerização Otimizada:** Criar um `Dockerfile` multi-stage, utilizando uma imagem base `distroless` ou `scratch`, garantindo que o processo rode como usuário *non-root*.

---

## Fase 3: Infraestrutura como Código (IaC) e FinOps
Toda a infraestrutura deve ser modular, testável e ter seus custos controlados antes de qualquer deploy.

- [ ] **1. Estrutura Modular:** Dividir o código do Terraform/OpenTofu em módulos claros (ex: `network`, `cluster_k8s`, `database`).
- [ ] **2. Cluster Local:** Provisionar um cluster Kubernetes local via IaC (K3s ou Kind) para validação dos manifestos.
- [ ] **3. Scan de IaC:** Utilizar o **Trivy** ou `tfsec` para escanear o código Terraform em busca de portas abertas indevidamente ou falta de criptografia.
- [ ] **4. FinOps:** Integrar a CLI do **Infracost** para avaliar as mudanças no Terraform e gerar um relatório de estimativa de custos na tela ou no Pull Request.
- [ ] **5. Gestão de Estado:** Configurar o backend remoto do Terraform com *state locking* (ex: S3 + DynamoDB na AWS, simulado via LocalStack).

---

## Fase 4: Integração e Entrega Contínuas (CI/CD & GitOps)
A esteira de deploy não deve ter intervenção humana para aplicar manifestos. O cluster K8s deve "puxar" as mudanças.

- [ ] **1. Pipeline de CI (GitHub Actions/GitLab CI):**
  - [ ] Lint do código da aplicação e do IaC.
  - [ ] Execução dos testes unitários e de integração.
  - [ ] Build da imagem Docker e push para um Container Registry.
  - [ ] Scan da imagem final gerada utilizando o Trivy.
- [ ] **2. Repositório de Manifestos:** Separar o código da aplicação dos manifestos Kubernetes (Kustomize ou Helm). O CI deve apenas atualizar a tag da imagem neste repositório.
- [ ] **3. Deploy do ArgoCD:** Instalar o ArgoCD no cluster e conectá-lo ao repositório de manifestos em modo de sincronização automática (*self-healing*).
- [ ] **4. Progressive Delivery:** Substituir o `Deployment` padrão do Kubernetes pelo `Rollout` do **Argo Rollouts** para implementar a estratégia Canary (ex: 10% -> pausa -> 50% -> 100%).

---

## Fase 5: Observabilidade Completa (O11y)
O sistema deve responder às perguntas: "Está lento?", "Está falhando?", "Por quê?".

- [ ] **1. Stack de Monitoramento:** Fazer o deploy do `kube-prometheus-stack` (Prometheus + Grafana + Alertmanager) via Helm.
- [ ] **2. Tracing Distribuído:** Instrumentar o código do microserviço com **OpenTelemetry** para rastrear o tempo de execução das requisições.
- [ ] **3. Centralização de Logs:** Fazer o deploy do **Grafana Loki** e configurá-lo para coletar os logs stdout/stderr de todos os pods do namespace.
- [ ] **4. Dashboards (Golden Signals):** Criar um dashboard no Grafana mostrando: Latência, Tráfego (RPS), Taxa de Erros e Saturação (CPU/Memória).
- [ ] **5. Dashboards (DORA Metrics):** Criar um dashboard focado em DevEx mostrando *Lead Time for Changes* e *Deployment Frequency*.

---

## Fase 6: Resiliência, Network Analysis e Chaos Engineering
Provar que a arquitetura e a observabilidade funcionam sob estresse e falhas reais.

- [ ] **1. Validação de Rede:** Utilizar ferramentas de análise de tráfego e pacotes (como `tcpdump`, `termshark` ou `traceroute` dentro de pods de debug) para inspecionar gargalos de CNI, roteamento interno e validar se as NetworkPolicies estão bloqueando o tráfego incorreto.
- [ ] **2. Engenharia de Caos:** Instalar o **LitmusChaos** ou **Chaos Mesh** no cluster.
- [ ] **3. Experimentos de Falha:**
  - [ ] *Pod Delete Chaos:* Matar pods aleatórios da aplicação para validar se o ReplicaSet recria rapidamente sem downtime percebido (Zero Downtime).
  - [ ] *Network Latency Chaos:* Injetar atrasos artificiais na comunicação com o banco de dados.
- [ ] **4. Rollback Automático:** Validar se os alertas do Prometheus disparam durante o experimento de caos e se o **Argo Rollouts** executa um rollback automático (abortar o Canary) ao detectar o aumento na latência ou erros.