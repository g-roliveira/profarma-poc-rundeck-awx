# Guia da PoC — ProFarma: Rundeck vs AWX

Este guia explica como executar os **mesmos playbooks Ansible** nos dois ambientes
(Rundeck e AWX), lado a lado, para demonstrar a diferença na camada de
**orquestração**.

---

## Estrutura do Repositório

```
profarma-poc-rundeck-awx/
├── playbooks/
│   ├── windows-transferencia-real.yml   ← Playbook Windows (WinRM)
│   ├── copy-files-scp.yml              ← Playbook Linux (SCP)
│   ├── transferencia-profarma.yml      ← Playbook genérico (AWX + Rundeck)
│   └── inventory/
│       └── windows.ini.example         ← Template de inventory
├── jobs/
│   ├── profarma-poc-job.yaml           ← Job Rundeck (4 etapas)
│   └── transferencia-arquivos.yaml     ← Orquestração com cron
├── docs/
│   └── GUIA-POC-PROFARMA.md           ← Este guia
└── k8s/
    ├── rundeck/                         ← Deploy Rundeck no k3s
    └── awx/                             ← Deploy AWX no k3s
```

---

## Pré-requisitos

### No servidor de orquestração (onde roda Rundeck ou AWX)

```bash
# Ansible + coleção Windows
ansible-galaxy collection install ansible.windows
pip install pywinrm
```

### No servidor Windows alvo

1. **WinRM habilitado** (já configurado pelo time ProFarma)
2. Firewall liberando porta **5985** (HTTP) ou **5986** (HTTPS)
3. Usuário com permissão de administrador local

---

## Configurar as Credenciais

### 1. Criar o arquivo de inventory

Copie o template e preencha com os dados reais:

```bash
cp playbooks/inventory/windows.ini.example playbooks/inventory/windows.ini
```

Edite `playbooks/inventory/windows.ini`:

```ini
[windows]
CXC-MAKER-DEV-0 ansible_host=10.10.100.132 ansible_user=administrador ansible_password=senha_real ansible_connection=winrm ansible_winrm_transport=ntlm ansible_winrm_server_cert_validation=ignore
```

> ⚠️ O arquivo `windows.ini` está no `.gitignore` — **nunca** será commitado.

---

## Opção A: Executar via Rundeck

### A.1 Acessar o Rundeck

```
URL: http://<servidor>:4440
Login: admin / admin
Projeto: Profarma
```

### A.2 Adicionar o nó Windows

No Rundeck, acesse o projeto **Profarma** → **Nodes** → **Add Node**:

```xml
<node name="CXC-MAKER-DEV-0"
      description="Windows Server ProFarma"
      hostname="10.10.100.132"
      osFamily="windows"
      username="administrador"
      winrm-authtype="basic"
      winrm-password-storage-path="keys/windows-password"
      winrm-no-ssl-cert-verification="true"
      node-executor="WinRMPython"
      file-copier="WinRMcpPython"
      tags="sync-server,windows,profarma" />
```

> A senha deve ser armazenada no **Key Storage** do Rundeck: `keys/windows-password`

### A.3 Criar o Job

1. **Jobs** → **New Job** → nome: `ProFarma - Transferencia Windows`
2. **Workflow** → adicionar 4 steps do tipo **Script**:

**Step 1 — Gerar e copiar arquivos:**
```bash
ansible-playbook /caminho/para/playbooks/windows-transferencia-real.yml \
  -i /caminho/para/playbooks/inventory/windows.ini \
  --tags copy,setup
```

**Step 2 — Validar checksum:**
```bash
ansible-playbook /caminho/para/playbooks/windows-transferencia-real.yml \
  -i /caminho/para/playbooks/inventory/windows.ini \
  --tags validate,report
```

**Step 3 — Auditoria:**
```bash
echo "====================================="
echo "ProFarma - AUDITORIA DE TRANSFERENCIA"
echo "Job: @job.name@ | Execucao: @job.execid@"
echo "Data: $(date -Iseconds)"
echo "====================================="
```

**Step 4 — Limpeza:**
```bash
echo "Limpeza concluida: $(date)"
```

3. **Schedule** → `*/10 * * * *` (a cada 10 minutos, demo)
4. **Notifications** → On Failure → Webhook/Email

### A.4 Executar

```bash
# Via UI: clique em "Run Job Now"
# Ou via API:
curl -u admin:admin -X POST \
  "http://localhost:4440/api/41/job/SEU_JOB_ID/executions"
```

### A.5 O que mostrar pro Weber no Rundeck

| Feature | Onde ver |
|---|---|
| **Dashboard** | Project → Activity |
| **Histórico de execuções** | Job → Executions |
| **Log passo a passo** | Execution → Log Output (cada step separado) |
| **Qual arquivo falhou** | Log mostra `❌ FALHA \| arquivo.txt \| motivo` |
| **Notificação** | Webhook disparado automaticamente |
| **Agendamento** | Job → Schedule |

---

## Opção B: Executar via AWX

### B.1 Acessar o AWX

```
URL: http://<servidor>:8052
Login: admin / (ver .secrets/awx)
```

### B.2 Criar o Inventory

1. **Inventories** → **Add** → nome: `ProFarma Windows`
2. **Groups** → **Add Group** → nome: `windows`
3. **Hosts** → **Add Host** → nome: `CXC-MAKER-DEV-0`
   - Variables (YAML):
   ```yaml
   ansible_host: 10.10.100.132
   ansible_user: administrador
   ansible_password: senha_real
   ansible_connection: winrm
   ansible_winrm_transport: ntlm
   ansible_winrm_server_cert_validation: ignore
   ```

### B.3 Criar a Credential

1. **Credentials** → **Add** → tipo: **Machine**
2. Nome: `ProFarma Windows Admin`
3. Username: `administrador`
4. Password: `senha_real`

### B.4 Criar o Project

1. **Projects** → **Add** → nome: `ProFarma PoC`
2. **SCM Type**: Git
3. **SCM URL**: `https://github.com/g-roliveira/profarma-poc-rundeck-awx.git`
4. **Branch**: `main`
5. ✅ **Update on Launch**

### B.5 Criar o Job Template

1. **Templates** → **Add** → **Job Template**
2. Nome: `ProFarma - Transferencia Windows`
3. **Job Type**: Run
4. **Inventory**: ProFarma Windows
5. **Project**: ProFarma PoC
6. **Playbook**: `playbooks/windows-transferencia-real.yml`
7. **Credentials**: ProFarma Windows Admin (Machine)
8. **Verbosity**: 2 (INFO)

### B.6 Lançar o Job

1. Templates → clique no foguete 🚀 ao lado do job template
2. Selecione a credencial → **Launch**

### B.7 O que mostrar pro Weber no AWX

| Feature | Onde ver |
|---|---|
| **Output do playbook** | Job → Output |
| **Histórico** | Jobs → lista de execuções |
| **Agendamento** | Templates → Schedules |

---

## Comparação Rundeck vs AWX

### Mesmo playbook, dois ambientes

| Critério | Rundeck | AWX |
|---|---|---|
| **Execução do playbook** | ✅ Igual | ✅ Igual |
| **Resultado** | 5/5 OK | 5/5 OK |
| **Tempo de execução** | ~10s (direto) | ~90s (sobe EE container) |

### Diferenciais do Rundeck

| Feature | Rundeck | AWX |
|---|---|---|
| **Dashboard de atividades** | ✅ Gráfico de sucesso/falha | ❌ Só lista |
| **Log step-by-step** | ✅ Cada etapa isolada | ❌ Output bruto do Ansible |
| **Gate condicional** | ✅ Step 1 falha → Step 2 nem roda | ✅ No playbook (YAML) |
| **Notificações** | ✅ Webhook, Email, Teams, Slack | ⚠️ Básico |
| **Visibilidade por arquivo** | ✅ Relatório com ✅/❌ por arquivo | ⚠️ Stdout bruto |
| **Key Storage** | ✅ Senhas criptografadas no cofre | ✅ Credentials |
| **Agendamento** | ✅ Cron granular (`*/5 8-18 * * 1-5`) | ✅ Simples |
| **Execução em grupo de nós** | ✅ Tags, filtros, grupos dinâmicos | ✅ Inventory groups |
| **Webhooks de entrada** | ✅ Dispara jobs via API externa | ✅ Callbacks |
| **ACL por projeto** | ✅ RBAC granular | ✅ RBAC |

### Cenário real — 300 scripts da ProFarma

| Problema do Weber | Rundeck resolve como? |
|---|---|
| "5 scripts por processo" | **Workflow**: junta tudo num job só |
| "Arquivo X falhou, qual?" | **Log step-by-step**: cada arquivo com ✅/❌ |
| "Preciso parar backup de log durante full" | **Gate condicional**: step só roda se anterior OK |
| "Usuário pergunta onde parou" | **Dashboard**: execução mostra etapa exata da falha |
| "Rodar a cada 15 min em horário comercial" | **Cron**: `*/15 8-18 * * 1-5` |
| "Notificar time quando falhar" | **Notification**: webhook/email/Teams automático |

---

## Troubleshooting

### WinRM: "Connection refused"

```bash
# Verificar se porta WinRM está aberta
Test-NetConnection -ComputerName localhost -Port 5985
```

### Rundeck: "No matched nodes"

Verifique se o nó tem o **node-executor** correto (`WinRMPython` para Windows).

### AWX: "no hosts matched"

Verifique se o host está no **grupo** `windows` dentro do inventory.

### AWX: Job fica "pending" eternamente

```bash
# Ver se o Execution Environment está sendo puxado
kubectl -n awx get pod -w | grep ee
```
