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
| Workflow multi-step | ✅ Nativo, com gate condicional entre steps | ⚠️ Workflow de jobs, menos granular |
| Execução condicional | ✅ "Step 1 falhou? Step 2 nem roda" | ⚠️ Via playbook (quando statement no YAML) |
| Multi-protocolo nativo | ✅ SSH, WinRM, SQL, HTTP, Script no mesmo job | ❌ Apenas Ansible (via Execution Environment) |
| Agendamento | ✅ Cron granular (`*/5 8-18 * * 1-5`) | ✅ Cron simples |
| Retry em falha | ✅ Configurável por step | ⚠️ Por job template |
| Notificações | ✅ Email, Teams, Slack, Webhook nativos | ⚠️ Callbacks limitados |

### 4.2 Dashboard, Reports e Histórico

| Critério | Rundeck | AWX |
|---|---|---|
| Dashboard visual | ✅ Gráfico de sucesso/falha por projeto | ❌ Não tem dashboard built-in |
| Drill-down por execução | ✅ Cada step com expandir/recolher | ❌ Output bruto do Ansible |
| Visibilidade de arquivo | ✅ Relatório com ✅/❌ por arquivo | ❌ Precisa parsear stdout |
| Relatório agendado (PDF/JSON) | ✅ Export nativo | ❌ Não tem |
| Gráfico de tendência | ✅ Success rate por job/projeto | ❌ Precisa de Grafana |
| Trilha de auditoria | ✅ Quem, quando, nó, resultado | ⚠️ Básico (quem executou) |

### 4.3 Escala e Cloud-Native

| Critério | Rundeck | AWX |
|---|---|---|
| Escala automática | ❌ Precisa configurar HPA manualmente | ✅ Operator com Instance Groups |
| Isolamento de execução | Processo local (shared) | ✅ Container por job (Execution Environment) |
| Deploy GitOps | ⚠️ Manual ou plugin de terceiros | ✅ Pull automático do Git |
| Multi-instância | ✅ Cluster mode (active/passive) | ✅ Operator gerencia |
| Execution Environments | ❌ Depende do SO do nó | ✅ Container isolado por job |

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

### Como cada ferramenta resolve

| Etapa do processo | Rundeck | AWX |
|---|---|---|
| 1. Copiar SAP → Sync Server Linux | Step 1 — SCP nativo | Job Template 1 — playbook Ansible |
| 2. Copiar Linux → Sync Server Windows | Step 2 — WinRM nativo | Job Template 2 — playbook Ansible |
| 3. Validar checksum | Step 3 — Script inline | Job Template 3 — playbook Ansible |
| 4. Se falhou, notificar | Step 4 condicional + Webhook | Workflow com notificação limitada |
| 5. Dashboard de falha | ✅ Mostra step e arquivo exato | ❌ Stdout bruto, precisa grep |
| 6. Reduzir 5 scripts → 1 job | ✅ 1 workflow Rundeck | ⚠️ 1 workflow AWX (menos granular) |

---

## 6. Recomendação

### Rundeck como orquestrador principal

Para o cenário da ProFarma — onde o problema é **orquestração de processos multi-etapa**
com necessidade de **visibilidade granular por arquivo** — o Rundeck entrega mais valor:

1. **Dashboard nativo** → Weber olha e sabe qual job, qual step, qual arquivo falhou
2. **Notificações** → integração direta com Teams/Email sem middleware
3. **Multi-protocolo** → SSH (Linux) e WinRM (Windows) no mesmo job, sem adaptação
4. **Agendamento granular** → `*/15 8-18 * * 1-5` nativo

### Onde o AWX complementa

Para execuções em **larga escala** (mesmo playbook em 100+ servidores simultâneos),
o AWX com Operator + Instance Groups + Execution Environments é superior em isolamento e paralelismo.

### Arquitetura híbrida (futuro)

```
Rundeck (orquestrador)
  │  Agenda, workflow condicional, notificações, dashboard
  │
  ├── Step 1: Copiar arquivos (SSH/WinRM nativo)
  ├── Step 2: Validar checksum (Script inline)
  ├── Step 3: Chamar AWX job template (execução em escala)
  └── Step 4: Notificar resultado (Webhook/Teams/Email)
```

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
