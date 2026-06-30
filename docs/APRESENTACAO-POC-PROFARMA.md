# ProFarma — Prova de Conceito: Rundeck vs AWX

**Data:** 30/06/2026
**Time 4Linux:** Gustavo Rodrigues, Samuel Gonçalves Pereira
**ProFarma:** Weber Muniz do Carmo

---

## 1. Objetivo da PoC

Validar uma solução de automação e orquestração para substituir os ~300 scripts Bash/PowerShell
que a ProFarma utiliza hoje para transferência de arquivos entre sistemas (SAP, Cobol, SQL Server),
com foco em:

1. **Execução real** de transferência de arquivos em servidor Windows
2. **Orquestração step-by-step** com tratamento de falha por arquivo
3. **Dashboard e histórico** com visibilidade granular
4. **Comparação lado a lado** entre AWX e Rundeck

---

## 2. Ambiente da PoC

| Componente | Detalhe |
|---|---|
| **Servidores** | Notebook local (k3s) + Windows Server real da ProFarma |
| **Windows Server** | `CXC-MAKER-DEV-0` (Windows Server 2022) |
| **Protocolo** | WinRM (NTLM) |
| **AWX** | v24.6.0 via Operator no k3s |
| **Rundeck** | v5.20.1 via Deployment no k3s (imagem customizada com Ansible 2.10.8) |
| **Playbooks** | Ansible (mesmo playbook executado nos dois ambientes) |

---

## 3. O que foi executado

### Playbook: `windows-transferencia-real.yml`

Pipeline de **4 etapas** executado contra o servidor Windows real:

| Etapa | Descrição | Resultado |
|---|---|---|
| **1/4** | Gerar 5 arquivos simulando SAP, Cobol e SQL Server | ✅ 5 arquivos (1-5MB cada) |
| **2/4** | Copiar e validar checksum MD5 de cada arquivo | ✅ 5/5 checksums OK |
| **3/4** | Relatório consolidado com visibilidade por arquivo | ✅ Relatório formatado |
| **4/4** | Gate condicional — se houver falha, aborta pipeline | ✅ Nenhuma falha |

### Resultado da execução (idêntico nos dois ambientes)

```
✅ SUCESSO | NF-e_loja_001.txt         | 1MB | SAP       | MD5=12B2FC56
✅ SUCESSO | estoque_filial_002.txt    | 2MB | Cobol     | MD5=380D73BB
✅ SUCESSO | precificacao_20260625.txt | 5MB | SQLServer | MD5=9B178CC6
✅ SUCESSO | log_vendas_diario.trn     | 3MB | SAP       | MD5=D7F62A9B
✅ SUCESSO | cadastro_produtos.csv     | 1MB | Cobol     | MD5=D4FF218B
─────────────────────────────────────────────────
TOTAL: 5 | OK: 5 | FALHAS: 0
```

---

## 4. Comparação: Rundeck vs AWX

### 4.1 Orquestração

| Critério | Rundeck | AWX |
|---|---|---|
| Workflow | ✅ Workflow nativo com steps individuais, gate condicional entre steps | ✅ Workflow Job Templates com ramificação por sucesso/falha |
| Granularidade natural | Step operacional — script, comando, API call | Job/playbook — a unidade é o playbook Ansible |
| Execução condicional | ✅ "Step 1 falhou? Step 2 nem roda" — direto e visível | ✅ Via ramificação de workflow — condicionais no nível de job |
| Multi-protocolo nativo | ✅ SSH, WinRM, SQL, HTTP, Script no mesmo job | ⚠️ Focado em Ansible — outros protocolos dependem de módulos/EE customizados |
| Agendamento | ✅ Cron granular (`*/5 8-18 * * 1-5`) | ✅ Cron via schedules |
| Notificações | ✅ Email, Teams, Slack, Webhook — nativos, por job | ✅ Webhook/callback — disponível, menos integrações diretas |

> **Diferença-chave:** Ambos têm workflow. O AWX trabalha no nível de **job/playbook**;
> o Rundeck desce ao nível de **step operacional** com drill-down visível.
> Para o caso ProFarma — "qual arquivo falhou em qual etapa?" — o step do Rundeck
> é a unidade natural de observação.

### 4.2 Dashboard, Reports e Histórico

| Critério | Rundeck | AWX |
|---|---|---|
| Dashboard visual | ✅ Gráfico de sucesso/falha por projeto, job e nó | ⚠️ Lista de execuções com status — funcional, sem gráfico built-in |
| Drill-down por execução | ✅ Cada step com expandir/recolher, output isolado | ✅ Tasks, hosts e eventos do Ansible — visível, mas contínuo |
| Visibilidade por arquivo | ✅ Relatório formatado com ✅/❌ por arquivo dentro do step | ⚠️ Depende do playbook modelar o output — senão vira leitura de log |
| Relatório agendado (PDF/JSON) | ✅ Export nativo agendável | ❌ Não disponível |
| Gráfico de tendência | ✅ Success rate por job/projeto | ❌ Precisa de ferramenta externa (Grafana) |
| Trilha de auditoria | ✅ Quem, quando, de onde, qual nó, resultado | ⚠️ Quem executou e resultado — menos granular |

> **Diferença-chave:** AWX expõe tasks e eventos do Ansible — é navegável, mas o
> output é contínuo. O Rundeck divide naturalmente por step, então "onde parou"
> é visual e imediato. Para "qual arquivo específico falhou", se o playbook não
> modelar isso com eventos individuais, no AWX vira grep no stdout.

### 4.3 Escala e Cloud-Native

| Critério | Rundeck | AWX |
|---|---|---|
| Escala automática | ❌ Precisa de HPA configurado manualmente | ✅ Operator com Instance Groups — escala nativa |
| Isolamento de execução | Processo compartilhado no container Rundeck | ✅ Container por job (Execution Environment) |
| Integração Git/SCM | ⚠️ Via plugin de terceiros ou script | ✅ Nativa — sync/update-on-launch de projetos Git |
| Multi-instância | ✅ Cluster mode (active/passive) | ✅ Operator gerencia réplicas |
| Execution Environments | ❌ Depende do SO e pacotes do nó | ✅ Container isolado e versionado por job |

### 4.4 Segurança e Governança

| Critério | Rundeck | AWX |
|---|---|---|
| Key Storage | ✅ Cofre criptografado (senhas, chaves SSH, tokens) | ✅ Credentials com criptografia |
| RBAC/ACL | ✅ Granular por projeto, job, nó | ✅ RBAC por organização/equipe |
| Integração LDAP/AD | ✅ Nativo (JAAS) | ✅ Via configuração |

---

## 5. Cenário ProFarma: 300 scripts

### Problema do Weber

> *"Tenho 5 scripts para um processo só. Um copia pra Linux, outro pra Windows, outro valida,
> outro manda pra outro servidor. Se um falha, tenho que descobrir qual arquivo parou."*

### Como cada ferramenta aborda

| Etapa do processo | Rundeck | AWX |
|---|---|---|
| 1. Copiar SAP → Sync Server Linux | Step 1 — SCP nativo | Job Template — playbook Ansible |
| 2. Copiar Linux → Sync Server Windows | Step 2 — WinRM nativo | Job Template — playbook Ansible |
| 3. Validar checksum | Step 3 — Script inline | Job Template — playbook Ansible |
| 4. Se falhou, notificar time | Step 4 condicional + Webhook/Email/Teams | Ramificação no workflow + webhook |
| 5. "Qual arquivo falhou?" | ✅ Step mostra ✅/❌ por arquivo | ⚠️ Depende do playbook modelar a saída |
| 6. Reduzir 5 scripts → 1 processo | ✅ 1 workflow Rundeck com 5 steps | ✅ 1 workflow AWX com 5 job templates |

> Ambos resolvem. A diferença está em quem olha: o Rundeck entrega o processo de
> negócio visível; o AWX entrega a execução Ansible padronizada.

---

## 6. Recomendação

### O que cada ferramenta resolve melhor

**O AWX Operator resolveu o problema de plataforma:** deploy declarativo, escala
automática via Instance Groups, isolamento de execução com Execution Environments
e integração nativa com Git/SCM. Para times que precisam executar playbooks Ansible
em escala, com inventory dinâmico, credentials centralizados e ambientes isolados,
o AWX moderno é a resposta certa.

**Mas o problema da ProFarma não é primariamente escala de playbook.** É
**orquestração e observabilidade operacional** de processos legados — ~300 scripts
que precisam virar workflows visíveis, com drill-down por etapa e arquivo.

### Por que Rundeck como orquestrador principal

1. **O processo de negócio fica visível.** Cada step do workflow é uma unidade
   operacional com output isolado — Weber olha o dashboard e sabe exatamente
   qual etapa falhou e qual arquivo parou, sem precisar interpretar stdout de playbook.

2. **Multi-protocolo sem adaptação.** SSH (Linux), WinRM (Windows), chamadas SQL,
   scripts shell, comandos HTTP — tudo no mesmo workflow, sem depender de módulos
   Ansible ou Execution Environments customizados.

3. **Notificações integradas ao negócio.** Email, Teams, Slack, Webhook saem
   direto do job, com contexto: qual step, qual nó, qual arquivo, qual horário.

4. **Dashboard e auditoria prontos.** Gráfico de tendência, relatório agendado,
   trilha de quem executou o quê — nativo, sem ferramenta externa.

### Onde o AWX complementa

Para playbooks Ansible complexos ou execuções em **larga escala** (100+ servidores
simultâneos), o AWX com Operator + Instance Groups + Execution Environments é
superior em isolamento, paralelismo e padronização da execução Ansible.

### Arquitetura recomendada

```
Quem olha:         Weber (negócio)                    Time técnico (infra)
                       │                                    │
Interface:        Rundeck Dashboard                    AWX Job Templates
                       │                                    │
Função:           Orquestração operacional             Execução Ansible escalável
                   Workflows multi-step                 Execution Environments
                   Notificações de negócio              Credentials centralizados
                   Auditoria e relatórios               Inventory dinâmico
                       │                                    │
                       └────────────┬───────────────────────┘
                                    │
                          Ansible Playbooks
                          (mesmo playbook, ambos ambientes)
                                    │
                          ┌─────────┴─────────┐
                     Windows Server    Linux Servers
                    (WinRM / SSH SCP)  (SSH nativo)
```

### Resumo

| Pergunta | Resposta |
|---|---|
| Qual é o orquestrador do processo? | **Rundeck** — mostra o negócio, step por step |
| Qual é o motor Ansible? | **AWX** — executa com EE, escala e isolamento |
| O Weber olha o quê? | **Rundeck** — dashboard com drill-down por arquivo |
| O time técnico usa o quê? | **AWX** — padronização e execução Ansible |
| Para a PoC, o foco é? | **Rundeck como orquestrador e dashboard** |

---

## 7. Entregáveis da PoC

| Item | Local |
|---|---|
| **Playbooks Ansible** | `playbooks/windows-transferencia-real.yml` |
| **Jobs Rundeck** | `jobs/` — pipeline 4 etapas com notificação |
| **Kustomize (Rundeck + AWX)** | `k8s/` — deploy completo no k3s |
| **Guia passo-a-passo** | `docs/GUIA-POC-PROFARMA.md` |
| **Apresentação** | Este documento |
| **Repositório** | GitHub (compartilhado com o time ProFarma) |

---

## 8. Próximos Passos

1. ✅ PoC validada — ambos os ambientes executam o mesmo playbook com sucesso
2. ⏭ Apresentar ao Weber a comparação lado a lado
3. ⏭ Se aprovado: definir escopo de implementação (fase 1: 10 workflows prioritários)
4. ⏭ Contrato e kickoff

---

*Documento preparado pela equipe 4Linux Consultoria.*
