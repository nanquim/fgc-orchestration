# FGC Orchestration

Repositório de orquestração da plataforma **FIAP Cloud Games**. Contém o `docker-compose.yml` global e os manifests Kubernetes unificados para subir todos os microsserviços em conjunto.

## Arquitetura de microsserviços

```
┌─────────────────────────────────────────────────────────────────┐
│                        FIAP Cloud Games                         │
│                                                                 │
│  ┌────────────┐    ┌─────────────┐    ┌──────────────┐         │
│  │  UsersAPI  │    │  CatalogAPI │    │  PaymentsAPI │         │
│  │  :5000     │    │  :5001      │    │  :5002       │         │
│  └─────┬──────┘    └──────┬──────┘    └──────┬───────┘         │
│        │                  │                  │                  │
│        └──────────────────┼──────────────────┘                  │
│                           │                                     │
│                    ┌──────▼──────┐                              │
│                    │  RabbitMQ   │                              │
│                    └──────┬──────┘                              │
│                           │                                     │
│                  ┌────────▼────────┐                            │
│                  │ NotificationsAPI│                            │
│                  │    :5003        │                            │
│                  └─────────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

## Fluxo de eventos

| Evento | Publicado por | Consumido por |
|--------|--------------|---------------|
| `UserCreatedEvent` | UsersAPI | NotificationsAPI |
| `OrderPlacedEvent` | CatalogAPI | PaymentsAPI |
| `PaymentProcessedEvent` | PaymentsAPI | CatalogAPI, NotificationsAPI |

## Repositórios

| Serviço | Repositório |
|---------|------------|
| UsersAPI | https://github.com/nanquim/fgc-users-api |
| CatalogAPI | https://github.com/nanquim/fgc-catalog-api |
| PaymentsAPI | https://github.com/nanquim/fgc-payments-api |
| NotificationsAPI | https://github.com/nanquim/fgc-notifications-api |
| Orchestration | https://github.com/nanquim/fgc-orchestration |

## Subir com Docker Compose

> Requer que as imagens já estejam buildadas ou disponíveis no registry.

### Build local de cada serviço

```bash
# Na pasta de cada repositório:
docker build -t nanquim/fgc-users-api:latest ../fgc-users-api
docker build -t nanquim/fgc-catalog-api:latest ../fgc-catalog-api
docker build -t nanquim/fgc-payments-api:latest ../fgc-payments-api
docker build -t nanquim/fgc-notifications-api:latest ../fgc-notifications-api
```

### Subir tudo

```bash
docker compose up
```

### Portas expostas

| Serviço | Porta |
|---------|-------|
| UsersAPI | http://localhost:5000 |
| CatalogAPI | http://localhost:5001 |
| PaymentsAPI | http://localhost:5002 |
| NotificationsAPI | http://localhost:5003 |
| RabbitMQ Management | http://localhost:15672 (guest/guest) |

## Deploy com Kubernetes

### Pré-requisitos

- Cluster Kubernetes (Minikube, Kind ou cloud)
- `kubectl` configurado
- Imagens publicadas no Docker Hub

### Aplicar manifests (ordem recomendada)

```bash
# 1. RabbitMQ compartilhado
kubectl apply -f k8s/rabbitmq/

# 2. Secrets e ConfigMaps de cada serviço
kubectl apply -f k8s/users-api/
kubectl apply -f k8s/catalog-api/
kubectl apply -f k8s/payments-api/
kubectl apply -f k8s/notifications-api/
```

### Verificar pods

```bash
kubectl get pods
kubectl get services
```

### Estrutura de manifests K8s

```
k8s/
├── rabbitmq/
│   └── deployment.yaml          # RabbitMQ Deployment + Service
├── users-api/
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── postgres.yaml            # PostgreSQL Deployment + Service + PVC
│   └── deployment.yaml          # UsersAPI Deployment + Service
├── catalog-api/
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── postgres.yaml
│   └── deployment.yaml
├── payments-api/
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── postgres.yaml
│   └── deployment.yaml
└── notifications-api/
    ├── configmap.yaml
    ├── secret.yaml
    └── deployment.yaml          # sem banco de dados
```

## Endpoints principais

### Autenticação (UsersAPI)

```bash
# Criar usuário
POST http://localhost:5000/users
{ "name": "João", "email": "joao@email.com", "password": "Senha@123" }

# Login (retorna JWT)
POST http://localhost:5000/auth/login
{ "email": "admin@fcg.com", "password": "Admin@123" }
```

### Catálogo (CatalogAPI) — requer JWT

```bash
# Listar jogos
GET http://localhost:5001/games

# Criar jogo (Admin)
POST http://localhost:5001/games
{ "title": "Cyber Quest", "description": "RPG", "price": 49.90 }

# Comprar jogo (User/Admin)
POST http://localhost:5001/games/{gameId}/purchase
```

### Pagamentos (PaymentsAPI) — requer JWT

```bash
# Consultar pagamento por orderId
GET http://localhost:5002/payments/order/{orderId}
```
