# FGC Orchestration — FIAP Cloud Games (Fase 3)

Repositório central de orquestração da plataforma **FIAP Cloud Games**.  
Contém o `docker-compose.yml` global e os manifests Kubernetes para subir toda a infraestrutura em conjunto.

---

## Arquitetura (Fase 3)

```mermaid
graph TD
    Cliente --> Kong["Kong API Gateway\n:8000"]

    Kong -->|POST /users, /auth| Users["UsersAPI\n:5010"]
    Kong -->|/games (JWT)| Catalog["CatalogAPI\n:5001"]

    Users -->|UserCreatedEvent| RabbitMQ
    Catalog -->|OrderPlacedEvent| Payments["PaymentsAPI\n:5002"]
    Payments -->|PaymentProcessedEvent| RabbitMQ

    RabbitMQ -->|user-created| AzureFn["Azure Function\nfgc-notifications-function"]
    RabbitMQ -->|payment-processed| AzureFn
    RabbitMQ -->|payment-processed| Catalog

    Catalog <-->|dados extendidos| MongoDB[("MongoDB\n:27017")]
    Catalog <-->|cache GET /games| Redis[("Redis\n:6379")]

    Prometheus["Prometheus\n:9090"] -->|scrape /metrics| Users
    Prometheus -->|scrape /metrics| Catalog
    Prometheus -->|scrape /metrics| Payments
    Grafana["Grafana\n:3000"] --> Prometheus
```

---

## Repositórios

| Serviço | Repositório |
|---|---|
| UsersAPI | https://github.com/nanquim/fgc-users-api |
| CatalogAPI | https://github.com/nanquim/fgc-catalog-api |
| PaymentsAPI | https://github.com/nanquim/fgc-payments-api |
| Notifications Function | https://github.com/nanquim/fgc-notifications-function |
| Orchestration | https://github.com/nanquim/fgc-orchestration |

---

## Como rodar localmente

### Pré-requisitos

- Docker Desktop com Docker Compose v2
- .NET 8+ SDK (para builds locais)
- [Azure Functions Core Tools v4](https://learn.microsoft.com/pt-br/azure/azure-functions/functions-run-local) (para a Function)

### 1. Build das imagens (se não disponíveis no registry)

```bash
docker build -t nanquim/fgc-users-api:latest    ../fgc-users-api
docker build -t nanquim/fgc-catalog-api:latest   ../fgc-catalog-api
docker build -t nanquim/fgc-payments-api:latest  ../fgc-payments-api
```

### 2. Subir infraestrutura completa

```bash
docker compose up
```

### 3. Subir a Azure Function (em outro terminal)

```bash
cd ../fgc-notifications-function
# edite local.settings.json com RabbitMQConnectionString=amqp://guest:guest@localhost:5672
func start
```

---

## Portas e URLs

| Serviço | URL |
|---|---|
| Kong Gateway (proxy) | http://localhost:8000 |
| Kong Admin API | http://localhost:8001 |
| UsersAPI (direto) | http://localhost:5010 |
| CatalogAPI (direto) | http://localhost:5001 |
| PaymentsAPI (direto) | http://localhost:5002 |
| RabbitMQ Management | http://localhost:15672 (guest/guest) |
| MongoDB | mongodb://localhost:27017 |
| Redis | localhost:6379 |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3000 (admin/admin) |

---

## Kong API Gateway

Todas as requisições externas devem passar pelo Kong na porta `8000`.

| Rota | Método | Destino | Auth |
|---|---|---|---|
| `/users` | POST | UsersAPI | Nenhuma |
| `/auth/login` | POST | UsersAPI | Nenhuma |
| `/games` | GET, POST | CatalogAPI | JWT obrigatório |
| `/games/{id}` | GET, PUT, DELETE | CatalogAPI | JWT obrigatório |
| `/games/{id}/purchase` | POST | CatalogAPI | JWT obrigatório |
| `/games/{id}/extended` | GET, PUT | CatalogAPI | JWT obrigatório |

### Exemplo de uso via Gateway

```bash
# 1. Criar usuário
curl -X POST http://localhost:8000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"João","email":"joao@email.com","password":"Senha@123"}'

# 2. Login (obter JWT)
TOKEN=$(curl -s -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@fcg.com","password":"Admin@123"}' | jq -r '.token')

# 3. Listar jogos (cacheado no Redis por 60s)
curl http://localhost:8000/games -H "Authorization: Bearer $TOKEN"

# 4. Buscar dados extendidos do MongoDB
curl http://localhost:8000/games/{id}/extended -H "Authorization: Bearer $TOKEN"
```

---

## Observabilidade — Prometheus + Grafana

### Dashboard FCG

Acesse http://localhost:3000 (admin/admin). O dashboard **FIAP Cloud Games — Observabilidade** é carregado automaticamente com 4 painéis:

| Painel | Query |
|---|---|
| Latência p95 por serviço | `histogram_quantile(0.95, ...)` |
| Requests por segundo | `rate(http_requests_received_total[5m])` |
| Taxa de erros 5xx | `rate(5xx) / rate(total)` |
| Requests por status code | `sum by (code)` |

### Métricas expostas

Todos os serviços expõem `/metrics` no formato Prometheus:
- `http_request_duration_seconds` — histograma de latência
- `http_requests_received_total` — contagem por método/status/handler

---

## Persistência Polígota

### MongoDB (dados extendidos de games)

Armazena informações não-relacionais por game: screenshots, tags, plataformas, rating, publisher.

```bash
# Inserir dados extendidos via endpoint (Admin)
curl -X PUT http://localhost:8000/games/{id}/extended \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "screenshots": ["https://cdn.fcg.com/game1-1.jpg"],
    "tags": ["RPG", "Multiplayer"],
    "platforms": ["PC", "PlayStation"],
    "averageRating": 4.7,
    "publisher": "FCG Studios",
    "releaseYear": 2024
  }'

# Consultar dados extendidos
curl http://localhost:8000/games/{id}/extended -H "Authorization: Bearer $TOKEN"
```

### Redis (cache de listagem de games)

`GET /games` usa cache com TTL de 60 segundos. Operações de escrita (POST/PUT/DELETE) invalidam o cache automaticamente.

---

## Serverless — Azure Function

A `NotificationsAPI` foi substituída por uma Azure Function (.NET 8 Isolated Worker).  
Repositório: `fgc-notifications-function`

| Função | Trigger | Ação |
|---|---|---|
| `UserCreatedFunction` | Fila `user-created` | Envia boas-vindas |
| `PaymentProcessedFunction` | Fila `payment-processed` | Envia confirmação de pagamento |

---

## Deploy com Kubernetes

### Aplicar manifests (ordem recomendada)

```bash
# Infraestrutura base
kubectl apply -f k8s/rabbitmq/
kubectl apply -f k8s/mongodb/
kubectl apply -f k8s/redis/

# Microsserviços
kubectl apply -f k8s/users-api/
kubectl apply -f k8s/catalog-api/
kubectl apply -f k8s/payments-api/

# API Gateway
kubectl apply -f k8s/kong/

# Observabilidade
kubectl apply -f k8s/observability/
```

### Estrutura K8s

```
k8s/
├── kong/               # API Gateway (deployment, service, configmap)
├── mongodb/            # MongoDB (deployment, pvc, service)
├── redis/              # Redis (deployment, service)
├── observability/      # Prometheus + Grafana (deployments, configmaps, services)
├── rabbitmq/           # RabbitMQ
├── users-api/          # UsersAPI + PostgreSQL
├── catalog-api/        # CatalogAPI + PostgreSQL
└── payments-api/       # PaymentsAPI + PostgreSQL
```

---

## Fluxo de Eventos

| Evento | Publicado por | Consumido por |
|---|---|---|
| `UserCreatedEvent` | UsersAPI | Azure Function (boas-vindas) |
| `OrderPlacedEvent` | CatalogAPI | PaymentsAPI |
| `PaymentProcessedEvent` | PaymentsAPI | CatalogAPI (library) + Azure Function (notificação) |
