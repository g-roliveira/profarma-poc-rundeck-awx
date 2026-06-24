# PoC Profarma — Rundeck + AWX no k3s

Stack de automação para a PoC da Profarma: **Rundeck** (orquestração) + **AWX** (execução Ansible) sobre Kubernetes (k3s).

## Arquitetura

```
k3s cluster
├── profarma-poc/
│   └── rundeck (5.20.1)     ← orquestrador de jobs
└── awx/
    ├── postgres-15           ← banco do AWX
    ├── awx-web (24.6.0)      ← interface web
    ├── awx-task (24.6.0)     ← scheduler + execução
    └── awx-operator (2.19)   ← operator do AWX
```

## Acessos

| Serviço | URL | Default Login |
|---------|-----|---------------|
| Rundeck | `http://rundeck.local` | `admin` / `admin` |
| AWX | `http://awx.local` | `admin` / ver `.secrets/awx` |

> Os hosts `rundeck.local` e `awx.local` precisam apontar para o IP do nó k3s (ajuste em `/etc/hosts` ou DNS).

## Pré-requisitos

- **k3s** ou Kubernetes 1.25+
- StorageClass `local-path` (padrão do k3s) ou qualquer outra
- 4GB+ RAM disponível, 10GB+ disco

## Deploy — Kustomize (recomendado)

```bash
# Deploy completo (Rundeck + AWX)
kubectl apply -k k8s/

# Apenas Rundeck
kubectl apply -k k8s/rundeck/

# Apenas AWX
kubectl apply -k k8s/awx/
```

O primeiro boot do AWX leva ~5min (migrações do banco).

## Deploy — Docker Compose (apenas Rundeck)

```bash
cp .env.dist .env
docker compose up -d
```

Rundeck disponível em `http://localhost:4440`.

## Estrutura do repositório

```
├── docker-compose.yml        # Rundeck standalone (Docker)
├── k8s/
│   ├── kustomization.yaml    # Raiz: aplica tudo
│   ├── rundeck/
│   │   ├── kustomization.yaml
│   │   └── base/             # namespace, pvc, deployment, svc, ingress
│   └── awx/
│       ├── kustomization.yaml # AWX Operator 2.19.0
│       └── awx-instance.yaml  # CR da instância AWX
├── .env.dist                 # Template de variáveis (Docker)
├── .gitignore                # .secrets/, .firecrawl/, .env
└── .secrets/                 # (gitignored) senhas geradas
```

## Senha do AWX

```bash
kubectl -n awx get secret awx-admin-password \
  -o jsonpath='{.data.password}' | base64 -d
```

## Destruir

```bash
kubectl delete -k k8s/
docker compose down --volumes   # se usou Docker
```

## Configuração adicional

### WinRM (nós Windows)

O projeto Rundeck `Profarma` já vem configurado com suporte WinRM. Cada nó Windows sobrescreve
o executor padrão SSH:

```xml
<node name="windows-server"
      osFamily="windows"
      username="admin"
      winrm-authtype="basic"
      winrm-password-storage-path="keys/windows-senha"
      node-executor="WinRMPython"
      file-copier="WinRMcpPython" />
```

### Customização dos hosts

Edite os Ingresses em:
- `k8s/rundeck/base/ingress.yaml` — host `rundeck.local`
- `k8s/awx/awx-instance.yaml` — `hostname: awx.local`
