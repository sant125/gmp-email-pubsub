# Arquitetura

## Visão geral

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                              │
│  ┌──────────┐  ┌──────────┐                                │
│  │  Pod 1   │  │  Pod 2   │                                │
│  │          │  │          │                                │
│  │ /metrics │  │ /metrics │                                │
│  └────┬─────┘  └────┬─────┘                                │
│       │             │                                       │
│       │             │                                       │
│  ┌────▼─────────────▼─────┐                                │
│  │  Prometheus Agent/      │                                │
│  │  GMP Collector          │                                │
│  └────────┬────────────────┘                                │
│           │                                                  │
└───────────┼──────────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────────┐
│   Managed Prometheus      │
│   (AMP / GMP)             │
│                           │
│  ┌─────────────────────┐  │
│  │  TSDB Storage       │  │
│  └─────────────────────┘  │
│                           │
│  ┌─────────────────────┐  │
│  │  Rules Engine       │  │
│  └─────────────────────┘  │
│                           │
│  ┌─────────────────────┐  │
│  │  AlertManager       │  │
│  └──────────┬──────────┘  │
└─────────────┼─────────────┘
              │
              ▼
     ┌────────────────┐
     │  SMTP / SNS    │
     └────────┬───────┘
              │
              ▼
         📧 Email
```

## Componentes

### 1. Application Pods

Seus apps rodando no Kubernetes. Precisam expor métricas no formato Prometheus:

```
GET http://pod-ip:port/metrics

# Retorna algo tipo:
http_requests_total{method="GET"} 1523
up{job="sample-app"} 1
container_memory_working_set_bytes 134217728
```

Qualquer app pode expor métricas. Tem bibliotecas client pra todas as linguagens:
- Go: `prometheus/client_golang`
- Python: `prometheus_client`
- Java: `io.prometheus.simpleclient`
- Node.js: `prom-client`

### 2. Prometheus Collector/Agent

**GCP (GMP Collector):**
- Roda como pods no namespace `gmp-system`
- Service discovery automático via `PodMonitoring` CRD
- Faz scraping dos pods e envia pro backend GMP
- Você não gerencia esses pods

**AWS (Prometheus Agent):**
- Você instala via Helm no cluster
- Configurado pra fazer remote write pro AMP
- Service discovery via `ServiceMonitor` CRD (Prometheus Operator)
- Você gerencia o deployment

O papel deles é:
1. Descobrir quais pods coletar
2. Scraping periódico (default: 30s)
3. Enviar métricas pro serviço gerenciado

### 3. Managed Prometheus (TSDB)

**GCP (Google Managed Prometheus):**
- Backend totalmente gerenciado
- Storage ilimitado (você paga por uso)
- 50GB/mês grátis
- Métricas ficam no Cloud Monitoring
- Query via Metrics Explorer ou API

**AWS (Amazon Managed Prometheus):**
- Backend totalmente gerenciado
- Storage pago por GB-mês
- 10M métricas grátis no free tier (primeiro ano)
- Query via grafana ou API compatível com Prometheus

Ambos armazenam as métricas em Time Series Database (TSDB), otimizado pra dados de séries temporais.

### 4. Rules Engine

Avalia as regras de alerta periodicamente:

```yaml
- alert: PodDown
  expr: up{job="sample-app"} == 0
  for: 1m
  labels:
    severity: critical
```

A cada intervalo (ex: 30s):
1. Roda a query `up{job="sample-app"} == 0`
2. Se der true, alerta vira PENDING
3. Se continuar true por 1min (`for`), vira FIRING
4. Notifica o AlertManager

O `for` evita alertas por ruído temporário (pod reiniciando, health check momentâneo, etc).

### 5. AlertManager

Recebe alertas do Rules Engine e decide o que fazer:

**Agrupamento (grouping):**
```yaml
route:
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 5m
```
Se 10 pods caírem em 10s, agrupa num alerta só.

**Roteamento:**
```yaml
routes:
- match:
    severity: critical
  receiver: pagerduty
- match:
    severity: warning
  receiver: email
```
Alertas critical vão pro PagerDuty, warning pro email.

**Silenciamento:**
Durante deploy, você pode silenciar alertas temporariamente.

**Retry:**
Se o envio falhar, tenta de novo.

**GCP:** AlertManager gerenciado (você não vê pod dele)

**AWS:** Você pode usar o AlertManager integrado do AMP ou instalar um no cluster.

### 6. Notification Channels

**GCP (SMTP direto):**
```yaml
receivers:
- name: email
  email_configs:
  - to: 'voce@email.com'
    smarthost: 'smtp.gmail.com:587'
    auth_username: 'seu@gmail.com'
    auth_password: 'app-password'
```

AlertManager conecta direto no SMTP e envia email.

**AWS (SNS):**
```yaml
receivers:
- name: sns
  sns_configs:
  - topic_arn: 'arn:aws:sns:...'
```

AlertManager publica no SNS, que distribui pra email/SMS/Lambda/etc.

## Fluxo completo de um alerta

```
T+0s    │ Pod crash
        │ Métrica up vira 0
        │
T+0s    │ Collector scrape
        │ Detecta up=0
        │ Envia pro Managed Prometheus
        │
T+30s   │ Rules Engine avalia
        │ Query: up{job="sample-app"} == 0
        │ Resultado: true
        │ Alerta → PENDING
        │
T+60s   │ Rules Engine avalia de novo
        │ Ainda true há 1min
        │ Alerta → FIRING
        │
T+61s   │ AlertManager recebe
        │ Inicia group_wait (10s)
        │ Aguarda outros alertas pra agrupar
        │
T+71s   │ AlertManager processa
        │ Agrupa alertas similares
        │ Aplica template de email
        │ Envia via SMTP/SNS
        │
T+72s   │ Gmail/SNS recebe
        │ Processa e entrega
        │
T+75s   │ 📧 Email chega
```

Tempo total: ~75 segundos desde o pod cair.

Se o pod voltar antes de 1min, o alerta nunca dispara (fica só em PENDING).

## Namespaces no GCP (GMP)

**gmp-system:**
- Onde o GMP coloca os pods gerenciados
- `collector`: scraper
- `rule-evaluator`: avalia regras
- `frontend`: API de query
- Você NÃO edita nada aqui

**gmp-public:**
- Onde VOCÊ coloca as configs
- `OperatorConfig`: config global
- `Rules`: regras de alerta
- `Secret` (alertmanager-config): config do AlertManager
- Rules SÓ funcionam se estiverem nesse namespace

**default (ou outro):**
- Onde seus apps rodam
- `PodMonitoring`: pode ficar aqui, junto com o app
- `Deployment`, `Service`, etc

Hierarquia:
```
┌─────────────────────┐
│    gmp-system       │ ← Pods gerenciados (read-only pra você)
└─────────────────────┘

┌─────────────────────┐
│    gmp-public       │ ← Suas configs (Rules, AlertManager)
└─────────────────────┘

┌─────────────────────┐
│    default          │ ← Seus apps + PodMonitoring
└─────────────────────┘
```

## Segurança e IAM

**GCP (Workload Identity):**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  annotations:
    iam.gke.io/gcp-service-account: my-app@project.iam.gserviceaccount.com
```

O pod usa a ServiceAccount K8s que mapeia pra ServiceAccount GCP. Consegue acessar recursos do GCP sem passar credenciais.

**AWS (IRSA - IAM Roles for Service Accounts):**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-app-role
```

O pod assume a IAM Role via OIDC. Consegue acessar recursos AWS (S3, SNS, etc) sem credenciais hardcoded.

## Anatomia de uma métrica

```
metric_name{label1="value1", label2="value2"} valor timestamp
```

Exemplo real:
```
http_requests_total{method="GET", path="/api/users", status="200"} 1523 1698765432
```

**metric_name:** O que você tá medindo

**labels:** Dimensões (pra filtrar, agrupar, etc)

**valor:** O número

**timestamp:** Quando foi coletado (opcional, Prometheus adiciona se não tiver)

### Tipos de métricas

**Counter:** Só aumenta (ou reseta a 0)
- `http_requests_total`
- `errors_total`
- Use `rate()` pra calcular taxa

**Gauge:** Sobe e desce
- `memory_usage_bytes`
- `active_connections`
- Use direto

**Histogram:** Distribui valores em buckets
- `http_request_duration_seconds`
- Usa `histogram_quantile()` pra calcular percentis

**Summary:** Como histogram, mas calcula quantis no cliente
- Raramente usado (histogram é melhor)

## PromQL: exemplos práticos

**Quantos pods tão up?**
```promql
up{job="sample-app"}
```

**Taxa de requisições por segundo (últimos 5min):**
```promql
rate(http_requests_total[5m])
```

**Uso de memória em %:**
```promql
(container_memory_working_set_bytes / container_spec_memory_limit_bytes) * 100
```

**P95 de latência:**
```promql
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

**Pods com uso de memória > 80%:**
```promql
(container_memory_working_set_bytes / container_spec_memory_limit_bytes) > 0.8
```

## Custos: breakdown

**GCP:**
- GKE Autopilot: ~$50-70/mês (depende do uso real)
- GKE Standard: ~$123/mês (nodes + control plane)
- GMP: $0 (até 50GB/mês)
- SMTP: $0 (Gmail grátis)
- **Total:** ~$50-123/mês

**AWS:**
- EKS: ~$73/mês (control plane) + nodes
- Nodes (2x t3.medium): ~$60/mês
- AMP: $0 no free tier, depois ~$2/mês
- SNS: ~$0.50/mês (email) ou ~$5/mês (SMS)
- **Total:** ~$133-145/mês

A parte de observabilidade em si é barata. O caro é o cluster.

## Managed vs Self-Hosted

**Managed:**
- Prós: Zero manutenção, escala automático, alta disponibilidade inclusa
- Contras: Custo, vendor lock-in, menos controle

**Self-Hosted:**
- Prós: Controle total, mais barato (se você já tem cluster)
- Contras: Você gerencia storage, escala, backup, HA, updates

Pra lab/aprendizado, managed é mais fácil.

Pra produção pequena (<1M métricas), self-hosted pode valer a pena.

Pra produção grande (>10M métricas), managed compensa.

## Limitações

**GCP (GMP):**
- Amarrado ao GKE (não funciona fora)
- Não dá pra acessar AlertManager direto (só via config)
- Query via console é menos flexível que Grafana

**AWS (AMP):**
- Precisa configurar remote write (mais complexo)
- Custo sobe rápido com muitas métricas
- Free tier só no primeiro ano

## Referências

- Prometheus docs: https://prometheus.io/docs/
- PromQL cheatsheet: https://promlabs.com/promql-cheat-sheet/
- GMP docs: https://cloud.google.com/stackdriver/docs/managed-prometheus
- AMP docs: https://docs.aws.amazon.com/prometheus/
- Awesome Prometheus Alerts: https://samber.github.io/awesome-prometheus-alerts/
