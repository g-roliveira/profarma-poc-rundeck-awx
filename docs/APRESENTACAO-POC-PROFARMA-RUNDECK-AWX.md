# PoC ProFarma: Rundeck e AWX

Apresentacao para demonstrar o ambiente atual, comparar as ferramentas e defender a recomendacao de uso para o cenario da ProFarma.

---

## 1. Objetivo da PoC

Validar como organizar e operar automacoes que hoje estao espalhadas em muitos scripts.

O problema real apresentado:

> "Tenho muitos scripts. Varios deles formam um processo. Quando falha, preciso saber em qual etapa parou e qual arquivo nao foi copiado."

Nesta PoC simulamos:

- 50 scripts agrupados em 5 processos.
- Execucao em Linux via SSH, usando o proprio node do k3s como alvo.
- Mesmo conceito replicado em Rundeck e AWX.
- Comparacao da experiencia operacional em cada ferramenta.

Observacao: o servidor Windows real estava indisponivel para automacao por WinRM no momento da PoC. Por isso, a validacao tecnica foi feita com SSH em Linux, mantendo o mesmo desenho de processo.

---

## 2. Ambiente da PoC

Cluster Kubernetes k3s:

- Node: `srv78las004`
- Ingress: Traefik
- AWX instalado via AWX Operator
- Rundeck instalado como deployment Kubernetes

Acessos da demo:

- AWX: `http://awx.local:30892`
- Rundeck: `http://rundeck.local:30892`

Componentes ativos:

- AWX Web
- AWX Task
- AWX Postgres
- AWX Operator
- Rundeck
- Traefik

---

## 3. O que foi configurado no Rundeck

Projeto:

```text
Profarma
```

Node SSH:

```text
srv78las004
```

Credencial:

```text
Key Storage: keys/suporte.password
```

Jobs principais:

```text
ProFarma/50 Scripts / ProFarma - Pipeline 50 Scripts
```

Jobs filhos:

```text
ProFarma - Processo 01 - Recebimento Fiscal
ProFarma - Processo 02 - Estoque SAP
ProFarma - Processo 03 - Precificacao Lojas
ProFarma - Processo 04 - Integracao Cobol
ProFarma - Processo 05 - Backup Log SQL
```

Cada processo simula 10 scripts.

O pipeline completo executa 50 scripts em sequencia.

---

## 4. Resultado validado no Rundeck

Execucao validada:

```text
ProFarma - Pipeline 50 Scripts
Status: succeeded
```

Exemplo de saida:

```text
OK - script_01 - recebimento-fiscal - arquivo=script_01_recebimento-fiscal.txt - host=srv78las004
OK - script_02 - recebimento-fiscal - arquivo=script_02_recebimento-fiscal.txt - host=srv78las004
...
OK - script_50 - backup-log-sql - arquivo=script_50_backup-log-sql.txt - host=srv78las004
ProFarma Pipeline 50 Scripts concluido
```

Ponto importante para demonstrar:

- O Rundeck mostra o processo como workflow.
- Cada etapa aparece separada.
- A falha fica localizada por etapa.
- O historico fica associado ao processo de negocio.

---

## 5. O que foi configurado no AWX

Inventory:

```text
ProFarma Linux SSH
```

Host:

```text
srv78las004
```

Credential:

```text
ProFarma Linux suporte SSH
```

Project:

```text
ProFarma PoC
```

Job Templates:

```text
ProFarma AWX - Processo 01 - Recebimento Fiscal
ProFarma AWX - Processo 02 - Estoque SAP
ProFarma AWX - Processo 03 - Precificacao Lojas
ProFarma AWX - Processo 04 - Integracao Cobol
ProFarma AWX - Processo 05 - Backup Log SQL
```

Workflow Job Template:

```text
ProFarma AWX - Workflow 50 Scripts
```

---

## 6. Resultado validado no AWX

Workflow executado:

```text
ProFarma AWX - Workflow 50 Scripts
Status: successful
```

Jobs executados com sucesso:

```text
ProFarma AWX - Processo 01 - Recebimento Fiscal
ProFarma AWX - Processo 02 - Estoque SAP
ProFarma AWX - Processo 03 - Precificacao Lojas
ProFarma AWX - Processo 04 - Integracao Cobol
ProFarma AWX - Processo 05 - Backup Log SQL
```

Ponto importante:

- O AWX executa bem os playbooks.
- O workflow do AWX encadeia job templates.
- A execucao fica mais proxima da visao tecnica Ansible.
- Para detalhes de arquivo/etapa, o usuario precisa consultar o output do playbook.

---

## 7. O que o AWX Operator melhorou

O AWX Operator moderniza a plataforma AWX.

| Area | Antes | Com AWX Operator |
|---|---|---|
| Deploy | Manual | Declarativo no Kubernetes |
| Upgrade | Mais manual | Gerenciado pelo Operator |
| Execucao | Ambiente mais fixo | Execution Environments |
| Isolamento | Menor | Container por job |
| Escala | Mais limitada | Melhor integracao com Kubernetes |
| Git | Manual ou menos integrado | Projeto sincronizado com SCM |
| Instance Groups | Limitado | Roteamento por grupos de execucao |

Mensagem principal:

> O Operator melhora muito a operacao do AWX como plataforma de execucao Ansible.

---

## 8. O que o AWX continua sendo

Mesmo com Operator, o AWX continua tendo o foco principal em Ansible:

- Inventories.
- Credentials.
- Projects Git.
- Job Templates.
- Workflow Job Templates.
- Execution Environments.
- Execucao em escala de playbooks.

Isso e muito forte quando o objetivo e:

- Rodar o mesmo playbook em muitos servidores.
- Padronizar execucao Ansible.
- Isolar dependencias em Execution Environments.
- Controlar inventories e credentials.

Mas o problema do Weber nao e apenas "executar playbook".

O problema e:

> "Como eu enxergo um processo operacional composto por varios scripts e sei exatamente onde falhou?"

---

## 9. Rundeck x AWX: comparacao honesta

| Criterio | Rundeck | AWX com Operator | Melhor para a ProFarma |
|---|---|---|---|
| Orquestracao operacional | Workflow step-by-step nativo | Workflow entre job templates | Rundeck |
| Visao por etapa | Natural na UI | Depende do output do playbook | Rundeck |
| Falha por arquivo | Facil modelar no step/log | Precisa parsear output Ansible | Rundeck |
| Dashboard de execucoes | Projeto, jobs, historico, activity | Jobs e workflow runs | Rundeck |
| Multi-protocolo | SSH, script, HTTP, SQL, plugins, APIs | Via modulo Ansible/EE | Rundeck |
| Notificacoes operacionais | Forte por job/step/status | Disponivel, mas mais tecnico | Rundeck |
| Execucao Ansible em escala | Possivel, mas nao e o foco | Excelente | AWX |
| Isolamento de dependencias | Depende do runner/node | Execution Environment | AWX |
| Git/SCM para playbooks | Possivel | Nativo no Project | AWX |
| Credentials | Key Storage | Credentials | Empate |

Conclusao da tabela:

> Para operar processos, Rundeck e mais claro. Para executar Ansible em escala, AWX e mais forte.

---

## 10. Por que Rundeck encaixa melhor no pedido do Weber

O Weber descreveu um cenario de 300 scripts.

O objetivo nao e necessariamente rodar 300 coisas em paralelo.

O objetivo e transformar:

```text
300 scripts soltos
```

em:

```text
50 workflows operacionais
```

Cada workflow pode representar um processo real:

- Recebimento fiscal.
- Estoque SAP.
- Precificacao.
- Integracao legado/Cobol.
- Backup de logs.
- Transferencia de arquivos.

No Rundeck, isso aparece como processo.

No AWX, isso aparece mais como execucao tecnica de playbook.

---

## 11. Exemplo da PoC: 50 scripts

Na PoC, criamos:

```text
1 pipeline pai
5 processos filhos
10 scripts por processo
50 scripts no total
```

No Rundeck:

```text
ProFarma - Pipeline 50 Scripts
  -> Processo 01 - Recebimento Fiscal
  -> Processo 02 - Estoque SAP
  -> Processo 03 - Precificacao Lojas
  -> Processo 04 - Integracao Cobol
  -> Processo 05 - Backup Log SQL
```

Isso representa exatamente o modelo desejado:

```text
varios scripts pequenos agrupados em processos de negocio
```

---

## 12. Como apresentar a demo no Rundeck

Abrir:

```text
Projeto: Profarma
Grupo: ProFarma/50 Scripts
Job: ProFarma - Pipeline 50 Scripts
```

Mostrar:

1. Workflow pai com 5 etapas.
2. Cada etapa chamando um processo filho.
3. Cada processo com 10 scripts.
4. Historico de execucoes.
5. Output com arquivo e host.
6. Status verde/vermelho por etapa.

Frase para usar:

> Aqui o Weber nao precisa procurar no meio de um log tecnico. Ele ve o processo, a etapa e o ponto de falha.

---

## 13. Como apresentar a demo no AWX

Abrir:

```text
Templates
Workflow Job Template: ProFarma AWX - Workflow 50 Scripts
```

Mostrar:

1. Workflow com 5 job templates.
2. Cada job template executando um playbook Ansible.
3. Inventory Linux SSH.
4. Credential SSH.
5. Output de uma execucao.

Frase para usar:

> O AWX executa muito bem o Ansible e organiza inventory, credentials e execution environments. Mas a visao aqui e mais tecnica, centrada no playbook.

---

## 14. A melhor arquitetura para a ProFarma

Recomendacao:

```text
Rundeck como orquestrador
AWX como executor Ansible
```

Fluxo ideal:

```text
Rundeck agenda o processo
      |
      v
Rundeck executa steps, gates e notificacoes
      |
      v
Quando precisar de Ansible, chama um Job Template no AWX
      |
      v
AWX executa com inventory, credential e execution environment
      |
      v
Rundeck recebe o resultado e registra no dashboard operacional
```

---

## 15. Quando usar Rundeck

Usar Rundeck para:

- Orquestrar processos com varios scripts.
- Criar workflows operacionais.
- Agendar rotinas.
- Dar visibilidade para operacao.
- Mostrar historico por processo.
- Executar scripts SSH, chamadas HTTP, SQL, automacoes simples e integracoes.
- Montar dashboards de sucesso/falha.

Exemplo ProFarma:

```text
Processo de transferencia diaria
  Step 1: parar servico
  Step 2: copiar arquivos
  Step 3: validar checksum
  Step 4: iniciar servico
  Step 5: notificar time
```

---

## 16. Quando usar AWX

Usar AWX para:

- Playbooks Ansible padronizados.
- Execucao em muitos servidores.
- Inventories complexos.
- Credentials centralizadas.
- Execution Environments.
- Separacao por Instance Groups.
- Integracao com Git para playbooks.
- Automacoes tecnicas versionadas.

Exemplo ProFarma:

```text
Aplicar baseline em 200 servidores
Atualizar configuracao de servico
Executar role Ansible padronizada
Provisionar ambiente com dependencias controladas
```

---

## 17. Mensagem executiva

O AWX Operator e uma evolucao importante.

Ele resolve muito bem:

- deploy,
- lifecycle,
- isolamento,
- escala,
- execucao Ansible moderna.

Mas ele nao muda o foco principal do AWX:

```text
AWX e um executor/controlador de Ansible.
```

Para o problema do Weber:

```text
300 scripts -> 50 processos rastreaveis
```

o Rundeck e mais adequado como camada de orquestracao.

---

## 18. Conclusao

Recomendacao para a PoC:

```text
Rundeck ganha como interface operacional e orquestrador.
AWX ganha como motor Ansible escalavel.
```

Recomendacao para arquitetura:

```text
Rundeck na frente
AWX por tras
```

Resumo:

- O operador olha o Rundeck.
- O time de automacao mantem playbooks no AWX.
- Os processos ficam claros.
- As falhas ficam rastreaveis.
- A ProFarma reduz complexidade operacional sem perder escala tecnica.

---

## 19. Roteiro rapido para a apresentacao

1. Abrir Rundeck.
2. Mostrar o projeto `Profarma`.
3. Abrir `ProFarma - Pipeline 50 Scripts`.
4. Mostrar os 5 processos filhos.
5. Abrir uma execucao bem-sucedida.
6. Mostrar os logs por script.
7. Abrir AWX.
8. Mostrar `ProFarma AWX - Workflow 50 Scripts`.
9. Mostrar que tambem executa com sucesso.
10. Explicar a diferenca:
    - AWX executa muito bem.
    - Rundeck explica melhor o processo.
11. Fechar com arquitetura hibrida.

---

## 20. Frase final

> Para a ProFarma, o ponto principal nao e trocar Ansible por Rundeck. E usar cada ferramenta no lugar certo: Rundeck para orquestrar e dar visibilidade; AWX para executar Ansible com escala e padronizacao.

