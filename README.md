# cloud-application-api

API de micro e-commerce para gestão de pedidos e itens, construída em **Python + FastAPI**, publicada na **Magalu Cloud** com deploy automatizado via **GitHub Actions**, banco **PostgreSQL gerenciado (DBaaS)** e stack de observabilidade **Prometheus + Grafana**.

> Projeto desenvolvido ao longo das Competências 3 a 6 do curso **Move Tech** — Magalu × Prósper Digital Skills (Formação em Cloud Computing).

---

## Sumário

- [Visão geral](#visão-geral)
- [Arquitetura](#arquitetura)
- [Endpoints](#endpoints)
- [Stack técnica](#stack-técnica)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Como rodar localmente](#como-rodar-localmente)
- [Infraestrutura na nuvem](#infraestrutura-na-nuvem)
- [Pipeline de CI/CD](#pipeline-de-cicd)
- [Banco de dados](#banco-de-dados)
- [Observabilidade](#observabilidade)
- [Resiliência](#resiliência)
- [Troubleshooting — problemas reais e soluções](#troubleshooting--problemas-reais-e-soluções)
- [Decisões de arquitetura (ADRs)](#decisões-de-arquitetura-adrs)
- [Próximos passos](#próximos-passos)

---

## Visão geral

A aplicação evoluiu em quatro etapas:

| Competência | Entrega |
|---|---|
| **3 — DevOps e Deploy** | Containerização, Container Registry, cluster K3s e pipeline de deploy |
| **4 — Dados e Persistência** | Banco PostgreSQL gerenciado (DBaaS) conectado via Kubernetes Secret |
| **5 — Observabilidade e Resiliência** | Prometheus + Grafana, ServiceMonitor, probes de liveness/readiness |
| **6 — Arquitetura de Soluções** | Diagrama C2, ADRs e análise de trade-offs (`docs/architecture.md`, `docs/adr/`) |

---

## Arquitetura

```mermaid
flowchart TB
    subgraph GH["GitHub"]
        Repo[Repositório] -->|push / workflow_dispatch| Actions[GitHub Actions]
    end

    subgraph MGC["Magalu Cloud — br-ne1"]
        Actions -->|docker push| Registry[(Container Registry)]
        Actions -->|kubectl apply| K3s

        subgraph VM["VM K3s — single node"]
            K3s[K3s API Server]
            Pod1[Pod cloud-application #1]
            Pod2[Pod cloud-application #2]
            SVC[Service LoadBalancer :80]
            SVC --> Pod1
            SVC --> Pod2
            Prom[Prometheus]
            Graf[Grafana]
            Prom -->|scrape /metrics via ServiceMonitor| SVC
            Graf -->|query PromQL| Prom
        end

        Pod1 -->|DATABASE_URL| PG[(PostgreSQL DBaaS)]
        Pod2 -->|DATABASE_URL| PG
    end

    User((Usuário)) -->|HTTP :80| SVC
    User -->|HTTP :NodePort| Graf
```

### Componentes

| Componente | Serviço MGC | Função |
|---|---|---|
| API | K3s (VM single node) — 2 réplicas | Processa as requisições HTTP |
| Banco de dados | DBaaS PostgreSQL | Persiste pedidos e itens |
| Imagens | Container Registry | Armazena versões da aplicação |
| Tráfego externo | Klipper ServiceLB (IP da VM, porta 80) | Distribui entre as réplicas e expõe acesso externo |
| Monitoramento | Prometheus + Grafana (Helm) | Coleta e visualiza métricas |
| CI/CD | GitHub Actions | Automatiza testes, build e deploy |

### Requisitos não-funcionais

| Requisito | Como medir | Alvo |
|---|---|---|
| Disponibilidade | Erros 5xx e uptime das probes no Grafana | 99,5% mensal |
| Latência | `histogram_quantile(0.95, ...)` do `/metrics` | P95 < 500 ms |
| Escalabilidade | Teste de carga (k6) + `rate(http_requests_total)` | 300 req/s sem degradar |

Detalhamento completo em [`docs/architecture.md`](docs/architecture.md).

---

## Endpoints

| Método | Rota | Descrição |
|---|---|---|
| `GET` | `/health` | Verifica se a API e o banco estão saudáveis — `{"status": "ok", "database": "ok"}` |
| `GET` | `/stats` | Estatísticas em tempo real (contagem de pedidos e itens) |
| `GET` | `/metrics` | Métricas no formato Prometheus |
| `GET` | `/docs` | Documentação interativa (Scalar) |
| `POST` | `/orders` | Cria um novo pedido |
| `GET` | `/orders` | Lista todos os pedidos |
| `GET` | `/orders/{id}` | Retorna um pedido com seus itens |
| `DELETE` | `/orders/{id}` | Cancela um pedido |
| `POST` | `/orders/{id}/items` | Adiciona um item ao pedido |
| `GET` | `/orders/{id}/items` | Lista os itens de um pedido |

---

## Stack técnica

- **Linguagem/Framework:** Python 3.11 + FastAPI
- **ORM:** SQLAlchemy
- **Banco:** PostgreSQL (produção) / SQLite (local, fallback sem `DATABASE_URL`)
- **Empacotamento:** Docker
- **Orquestração:** Kubernetes (K3s)
- **CI/CD:** GitHub Actions
- **Observabilidade:** Prometheus, Grafana, kube-prometheus-stack (Helm)
- **Cloud:** Magalu Cloud (Virtual Machine, Container Registry, Kubernetes/K3s, DBaaS)

---

## Estrutura do repositório

```
.
├── app/                        # Código-fonte da aplicação (FastAPI)
├── docs/
│   ├── data-model.md            # Modelagem de dados (Competência 4)
│   ├── architecture.md          # Diagrama C2, componentes e trade-offs (Competência 6)
│   └── adr/
│       ├── 001-kubernetes-deploy.md
│       └── 002-dbaas-postgresql.md
├── k8s/
│   ├── app.yaml                 # Deployment + Service
│   └── servicemonitor.yaml      # ServiceMonitor para o Prometheus
├── tests/                       # Testes automatizados
├── .github/workflows/deploy.yml # Pipeline de CI/CD
├── Dockerfile
├── docker-compose.yml
└── pyproject.toml
```

---

## Como rodar localmente

**Pré-requisito:** [Docker Desktop](https://www.docker.com/products/docker-desktop/) ou [Docker Engine](https://docs.docker.com/engine/install/).

```bash
docker compose up --build
```

Acesse a documentação interativa em `http://localhost:8000/docs`.

Sem `DATABASE_URL` configurada, a aplicação usa SQLite local — os dados são perdidos ao reiniciar o container (comportamento esperado fora de produção).

---

## Infraestrutura na nuvem

| Item | Valor |
|---|---|
| Região | `br-ne1` (Nordeste) |
| Cluster | K3s single-node, provisionado via [`k3s-mgc`](https://github.com/move-tech-cloud-computing/k3s-mgc) |
| Tipo de VM | `BV2-2-10` → redimensionada para maior capacidade de RAM após saturação de memória (ver [Troubleshooting](#troubleshooting--problemas-reais-e-soluções)) |
| Security Group | `sg-k3s-cluster` — portas liberadas: `22` (SSH), `80` (HTTP), `6443` (API Kubernetes), `8000` (API direta), `3000`/`9090` (observabilidade), NodePort do Grafana |
| Container Registry | `container-registry.br-ne1.magalu.cloud/<nome-do-registry>` |

Provisionamento automatizado com:

```bash
~/k3s.sh kubernetes cluster create --name cloud-application-cluster
~/k3s.sh kubernetes cluster configure-registry --cluster-id <ID>
~/k3s.sh kubernetes cluster kubeconfig --cluster-id <ID> --raw > kubeconfig.yaml
```

---

## Pipeline de CI/CD

Definido em [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml), disparado manualmente (`workflow_dispatch`):

1. **test** — instala dependências e roda os testes automatizados (`pytest`)
2. **build-and-deploy** (só se os testes passarem):
   - Login no Container Registry da MGC
   - Build e push da imagem Docker
   - Configuração do `kubectl` a partir do Secret `MGC_KUBECONFIG`
   - Criação/atualização do Secret `db-secret` a partir de `DATABASE_URL`
   - Aplicação dos manifests (`envsubst` injeta `MGC_REGISTRY_NAME`) e espera do rollout

### Secrets necessários (GitHub Actions)

| Secret | Descrição |
|---|---|
| `MGC_REGISTRY_USER` | Usuário do Container Registry |
| `MGC_REGISTRY_PASSWORD` | Senha do Container Registry |
| `MGC_REGISTRY_NAME` | Nome do registry na MGC |
| `MGC_KUBECONFIG` | Conteúdo do `kubeconfig.yaml` do cluster |
| `DATABASE_URL` | String de conexão PostgreSQL (`postgresql://usuario:senha@host:5432/orders`) |

---

## Banco de dados

- PostgreSQL gerenciado (DBaaS) na Magalu Cloud.
- O banco `orders` é criado manualmente no console do DBaaS (não é criado automaticamente); as **tabelas** (`orders`, `items`) são criadas automaticamente pela aplicação via `Base.metadata.create_all()`.
- A conexão é injetada no pod via Kubernetes Secret (`db-secret`), nunca exposta em código ou manifests.
- Modelagem completa em [`docs/data-model.md`](docs/data-model.md).

---

## Observabilidade

Stack `kube-prometheus-stack` instalada via Helm no namespace `monitoring`:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.service.type=LoadBalancer \
  --set prometheus.service.type=LoadBalancer \
  --set grafana.adminPassword=admin
```

- **Grafana:** acessível pelo IP da VM na porta exposta pelo Service (NodePort, já que a porta 80 é usada pela aplicação — ver [Troubleshooting](#troubleshooting--problemas-reais-e-soluções))
- **Prometheus:** `http://<IP-DA-VM>:9090` → **Status → Targets** confirma `cloud-application` como `UP`
- **ServiceMonitor:** [`k8s/servicemonitor.yaml`](k8s/servicemonitor.yaml) conecta o Prometheus ao endpoint `/metrics` da aplicação, usando o label obrigatório `release: monitoring`

Logs estruturados em JSON, filtráveis via:

```bash
kubectl logs -l app=cloud-application -f
kubectl logs -l app=cloud-application | grep order_created
```

---

## Resiliência

O `k8s/app.yaml` define **liveness** e **readiness probes** sobre `/health`:

- **Liveness:** se `/health` falhar repetidamente, o Kubernetes reinicia o pod
- **Readiness:** se `/health` falhar, o pod para de receber tráfego até se recuperar

Testado com:

```bash
kubectl delete pod <nome-do-pod>       # Kubernetes recria automaticamente
kubectl scale deployment cloud-application --replicas=0   # derruba tudo
kubectl scale deployment cloud-application --replicas=2   # restaura
```

---

## Troubleshooting — problemas reais e soluções

Registro dos principais problemas enfrentados durante o deploy, para referência futura.

### 1. Container Registry: "Um registro com esse nome já existe"
Nomes de registry são **globalmente únicos** na Magalu Cloud (como buckets S3). Nomes genéricos (`cloud-application-registry`) colidem com outras contas. **Solução:** usar um nome mais específico (ex: prefixado com usuário/projeto).

### 2. Aviso de host SSH alterado (`REMOTE HOST IDENTIFICATION HAS CHANGED`)
Ocorre quando a VM é recriada — o fingerprint SSH muda. **Solução:**
```bash
ssh-keygen -f '~/.ssh/known_hosts' -R '<IP>'
```

### 3. Script `k3s.sh` com opções de VM divergentes do enunciado
O repositório do script evolui rapidamente; as opções de máquina disponíveis via prompt interativo podem não incluir exatamente o tipo citado na atividade (`BV2-2-40`). **Solução:** escolher a opção mais próxima em vCPU/RAM disponível no momento (ex: `BV2-2-10`).

### 4. Cluster criado na região errada
A região é definida no perfil da `mgc` CLI, não no script de criação do cluster. **Solução:**
```bash
mgc config get region
mgc config set region br-ne1
```
Recursos de regiões diferentes são isolados — é necessário recriar cluster/registry na região correta caso já tenham sido criados na região errada.

### 5. `ImagePullBackOff` / `no basic auth credentials`
O K3s não tem credenciais para autenticar no Container Registry privado. **Solução:** criar o secret de autenticação e associá-lo ao service account:
```bash
kubectl create secret docker-registry mgc-registry-secret \
  --docker-server=container-registry.br-ne1.magalu.cloud \
  --docker-username=<usuario> --docker-password=<senha>
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "mgc-registry-secret"}]}'
```

### 6. `curl` trava sem resposta (`Trying <IP>:80...`)
Porta bloqueada no Security Group — não é erro de aplicação. O script `k3s.sh` só libera `22`, `6443` e `8000` por padrão. **Solução:** abrir a porta manualmente:
```bash
mgc network security-groups rules create \
  --security-group-id=<SG-ID> --direction=ingress --ethertype=IPv4 \
  --protocol=tcp --port-range-min=80 --port-range-max=80 \
  --remote-ip-prefix=0.0.0.0/0
```

### 7. `FATAL: database "orders" does not exist`
O PostgreSQL não cria o banco automaticamente — só as tabelas são criadas pela aplicação (`create_all()`). **Solução:** criar o banco manualmente no console do DBaaS antes do primeiro deploy com `DATABASE_URL` configurada.

### 8. Grafana com `EXTERNAL-IP` em `<pending>`
Dois serviços `LoadBalancer` (aplicação e Grafana) competindo pela porta 80 no mesmo node — o ServiceLB do K3s não consegue alocar a porta duas vezes. **Diagnóstico:**
```bash
kubectl describe pod -n kube-system svclb-monitoring-grafana-... 
# Events: "didn't have free ports for the requested pod ports"
```
**Solução aplicada:** acessar o Grafana pelo NodePort alocado automaticamente (ex: `http://<IP-DA-VM>:<NodePort>`) em vez de esperar por um IP externo dedicado na porta 80.

### 9. Grafana carregando infinitamente + `401 Unauthorized` em `/api/login/ping`
Sessão não autenticada. **Solução:** acessar `/login` diretamente e usar as credenciais configuradas na instalação do Helm (`admin` / senha definida em `grafana.adminPassword`), ou recuperar via:
```bash
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d
```

### 10. `TLS handshake timeout` ao rodar `kubectl`
VM saturada de memória (RAM 100% utilizada, sem swap) rodando K3s + aplicação + stack completo de monitoramento (Prometheus, Grafana, Alertmanager) em uma VM de baixa capacidade. **Diagnóstico:**
```bash
free -h   # available próximo de 0
systemctl status k3s   # uso de memória do processo k3s-server elevado
```
**Solução:** aumento da capacidade de RAM da VM via console da Magalu Cloud.

### 11. ServiceMonitor criado, mas Prometheus não descobre o target
O `ServiceMonitor` seleciona um `Service` por **label** (`app: cloud-application`) e por **nome de porta** (`http`) — não basta o Deployment ter esses atributos. **Causa raiz:** o `Service` no `k8s/app.yaml` não tinha `labels` nem `name: http` na porta. **Solução:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: cloud-application
  labels:
    app: cloud-application   # necessário para o selector do ServiceMonitor
spec:
  type: LoadBalancer
  selector:
    app: cloud-application
  ports:
  - name: http                # necessário: ServiceMonitor referencia a porta pelo nome
    port: 80
    targetPort: 8000
```

### 12. `mgc config set` retornando erro de flag
A sintaxe da CLI é posicional, não `key=value`:
```bash
# Errado
mgc config set region=br-ne1
# Correto
mgc config set region br-ne1
```

### 13. `kubectl` sem `KUBECONFIG` cai em `localhost:8080`
Sintoma de que o arquivo de kubeconfig não existe no caminho esperado ou não foi exportado na sessão atual. **Solução:**
```bash
~/k3s.sh kubernetes cluster kubeconfig --cluster-id <ID> --raw > ~/kubeconfig.yaml
export KUBECONFIG=~/kubeconfig.yaml
```

### 14. SSH `Permission denied (publickey)` para a VM do cluster
A chave usada pelo script para provisionar o cluster pode não ser a chave SSH padrão (`id_ed25519`) da máquina cliente. **Solução:** identificar a chave certa (`mgc profile ssh-keys list` + comparar com `~/.ssh/*.pub`) e especificá-la explicitamente:
```bash
ssh -i ~/.ssh/<chave-correta> ubuntu@<IP-da-VM>
```
> Nunca usar `sudo ssh` — as chaves ficam no `$HOME` do usuário, não do root.

---

## Decisões de arquitetura (ADRs)

| ADR | Decisão |
|---|---|
| [001 — Kubernetes para deploy](docs/adr/001-kubernetes-deploy.md) | K3s em VM única, em vez de MKS gerenciado — custo menor e provisionamento mais rápido |
| [002 — DBaaS PostgreSQL](docs/adr/002-dbaas-postgresql.md) | Banco gerenciado, em vez de PostgreSQL em pod — backup e HA sem custo operacional |

Tabela completa de trade-offs em [`docs/architecture.md`](docs/architecture.md#trade-offs).

---

## Próximos passos

| Melhoria | Por quê |
|---|---|
| HTTPS / TLS | Toda API em produção deve ser acessada por HTTPS |
| Autoscaler (HPA) | Escala automaticamente conforme a carga |
| Versionamento de API (`/v1/orders`) | Evolução sem quebrar clientes |
| Rate limiting | Protege o banco de sobrecargas |
| Cache (Redis) | Reduz consultas repetidas ao banco |
| Migrações de schema (Alembic) | Controle de versão do banco |
| Testes de carga (k6) | Valida comportamento sob alto tráfego |
| Migrar para MKS | Alta disponibilidade real — manifests já são compatíveis |

---

> Projeto desenvolvido como parte do curso **Move Tech** — Magalu × Prósper Digital Skills.
> Inspirado no tutorial [Construindo APIs robustas utilizando Python](https://github.com/luizalabs/tutorial-python-brasil), do LuizaLabs.