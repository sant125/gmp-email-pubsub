# Arquitetura

## Componentes

### 1. Kubernetes Cluster (EKS/GKE)
Roda os pods da aplicação

### 2. Sample App (Pods)
Expõe métricas em `/metrics`:
- `up` - status (1=up, 0=down)
- `http_requests_total` - total requests
- `container_memory_usage_bytes` - uso memória
- `container_cpu_usage_seconds_total` - uso CPU

### 3. Prometheus Agent (AWS) / GMP Collector (GCP)
- Service Discovery: encontra pods
- Scraping: coleta métricas (default: 30s)
- Remote Write: envia pro serviço gerenciado

### 4. Managed Prometheus
- TSDB: armazena métricas
- Rules Engine: avalia regras
- AlertManager: processa alertas

### 5. Notification Service (SNS/Pub/Sub)
Distribui notificações

### 6. Email/SMS
Entrega final

## Fluxo de Alerta

```
1. Pod crash → up=0
2. Prometheus scrape → detecta up=0
3. Rules engine avalia → alerta PENDING
4. Após 1min ainda 0 → alerta FIRING
5. AlertManager recebe → agrupa/deduplica
6. Envia pro SNS/Pub/Sub
7. Email/SMS enviado
```

## Segurança

**AWS (IRSA):**
- ServiceAccount K8s vinculado a IAM Role
- Pod assume role automaticamente via OIDC

**GCP (Workload Identity):**
- ServiceAccount K8s vinculado a Service Account GCP
- Pod usa credenciais GCP automaticamente

## Custos

- **AWS:** ~$145/mês (free tier: $10-15/mês)
- **GCP:** ~$122/mês (observabilidade grátis no free tier)

Ver COSTS.md pra detalhes.
