# Fontes de Métricas no Kubernetes com GMP

## TL;DR

**Você NÃO precisa que sua aplicação exponha `/metrics` para monitorar CPU, memória e estado dos pods.**

Essas métricas já existem automaticamente via **kubelet** e **kube-state-metrics**, que o GMP coleta por padrão.

Você só precisa de `/metrics` na app se quiser métricas de negócio (requests/s, latência, erros, filas, vendas, etc).

## Fontes de Métricas

### 1. Métricas do Kubelet (cAdvisor)

**O que é:**
- cAdvisor roda dentro do kubelet em cada node
- Coleta métricas de **recursos** dos containers via cgroups
- GMP coleta essas métricas **automaticamente**

**Métricas disponíveis:**
```promql
# CPU
container_cpu_usage_seconds_total
container_cpu_cfs_throttled_seconds_total

# Memória
container_memory_working_set_bytes
container_memory_usage_bytes
container_spec_memory_limit_bytes

# Disco
container_fs_usage_bytes
container_fs_limit_bytes

# Rede
container_network_receive_bytes_total
container_network_transmit_bytes_total
```

**Precisa de PodMonitoring?** ❌ **NÃO!**

Essas métricas já existem no GMP sem você fazer nada.

**Exemplo de alerta usando métricas do kubelet:**
```yaml
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: resource-alerts
  namespace: default
spec:
  groups:
  - name: resources
    interval: 30s
    rules:
    # Memória > 80%
    - alert: HighMemoryUsage
      expr: |
        (container_memory_working_set_bytes{container!=""}
        / container_spec_memory_limit_bytes{container!=""}) > 0.8
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Container usando muita memória"
        description: "{{ $labels.pod }}/{{ $labels.container }}: {{ $value | humanizePercentage }}"

    # CPU throttling
    - alert: CPUThrottling
      expr: |
        rate(container_cpu_cfs_throttled_seconds_total[5m]) > 0.5
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Container com CPU throttling"
        description: "{{ $labels.pod }} está sendo throttled"
```

**Nenhum `/metrics` necessário na app!**

### 2. Métricas do kube-state-metrics

**O que é:**
- Serviço que lê a API do Kubernetes
- Exporta métricas sobre o **estado** dos recursos (Pods, Deployments, Services, etc)
- GMP coleta essas métricas **automaticamente**

**Métricas disponíveis:**
```promql
# Estado dos Pods
kube_pod_status_phase{phase="Running|Pending|Failed|Succeeded"}
kube_pod_container_status_ready
kube_pod_container_status_restarts_total

# Deployments
kube_deployment_spec_replicas
kube_deployment_status_replicas_available
kube_deployment_status_replicas_unavailable

# Nodes
kube_node_status_condition{condition="Ready|MemoryPressure|DiskPressure"}

# Resources requests/limits
kube_pod_container_resource_requests{resource="cpu|memory"}
kube_pod_container_resource_limits{resource="cpu|memory"}
```

**Precisa de PodMonitoring?** ❌ **NÃO!**

O GMP já scrape o kube-state-metrics automaticamente.

**Exemplo de alertas usando kube-state-metrics:**
```yaml
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: k8s-state-alerts
  namespace: default
spec:
  groups:
  - name: kubernetes_state
    interval: 30s
    rules:
    # Pod não está Running
    - alert: PodNotRunning
      expr: |
        kube_pod_status_phase{phase!="Running", phase!="Succeeded"} == 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod em estado {{ $labels.phase }}"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} está {{ $labels.phase }}"

    # Deployment com réplicas insuficientes
    - alert: DeploymentReplicasMismatch
      expr: |
        kube_deployment_spec_replicas
        !=
        kube_deployment_status_replicas_available
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Deployment com réplicas faltando"
        description: "{{ $labels.namespace }}/{{ $labels.deployment }}: esperado {{ $value }}, disponível menos"

    # Pod crashlooping
    - alert: PodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod reiniciando frequentemente"
        description: "{{ $labels.namespace }}/{{ $labels.pod }} está crashlooping"
```

**Nenhum `/metrics` necessário na app!**

### 3. Métricas da Aplicação (Custom Metrics)

**O que é:**
- Métricas que **apenas sua aplicação sabe**
- Precisa instrumentar o código (biblioteca Prometheus client)
- Precisa expor endpoint `/metrics`
- Precisa criar `PodMonitoring` pra GMP coletar

**Métricas disponíveis (exemplos):**
```promql
# HTTP
http_requests_total{method="GET|POST", status="200|500"}
http_request_duration_seconds_bucket  # Histograma de latência
http_requests_in_flight_gauge

# Negócio
myapp_orders_processed_total
myapp_payment_failures_total
myapp_active_users_gauge
myapp_queue_size

# Database
myapp_db_query_duration_seconds
myapp_db_connections_active

# Custom
myapp_feature_flags_enabled{flag="feature_x"}
myapp_background_jobs_total{status="success|failure"}
```

**Precisa de PodMonitoring?** ✅ **SIM!**

```yaml
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: myapp-monitoring
  namespace: default
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics  # Port name do Service
    interval: 30s
    path: /metrics
```

**Exemplo de app instrumentada (Go):**
```go
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "status"},
    )

    ordersProcessed = prometheus.NewCounter(
        prometheus.CounterOpts{
            Name: "orders_processed_total",
            Help: "Total orders processed",
        },
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal)
    prometheus.MustRegister(ordersProcessed)
}

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        httpRequestsTotal.WithLabelValues(r.Method, "200").Inc()
        w.Write([]byte("OK"))
    })

    http.HandleFunc("/process-order", func(w http.ResponseWriter, r *http.Request) {
        // Processa pedido...
        ordersProcessed.Inc()
        w.Write([]byte("Order processed"))
    })

    // Expõe /metrics
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
}
```

### 4. Métrica especial: `up`

A métrica `up` é **gerada pelo Prometheus** (não pela app) durante o scrape.

**Como funciona:**
```
Prometheus/GMP faz scrape → Sucesso → up{job="..."} = 1
Prometheus/GMP faz scrape → Falha   → up{job="..."} = 0
```

**Quando é útil:**
- Detectar se a app parou de responder (porta fechada, app crashed)
- Diferente de detectar pod down (que usa kube-state-metrics)

**Exemplo:**
```yaml
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: scrape-alerts
  namespace: default
spec:
  groups:
  - name: scraping
    interval: 30s
    rules:
    # App não está respondendo ao scrape
    - alert: ScrapeTargetDown
      expr: sum(up{job="default/myapp-monitoring"}) < 1
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "App não está respondendo"
        description: "{{ $value }} targets ativos (esperado >= 1)"
```

## Comparação: Quando Usar Cada Fonte

| Tipo de Métrica | Fonte | Precisa /metrics? | Exemplo |
|----------------|-------|-------------------|---------|
| CPU do pod | kubelet (cAdvisor) | ❌ Não | `container_cpu_usage_seconds_total` |
| Memória do pod | kubelet (cAdvisor) | ❌ Não | `container_memory_working_set_bytes` |
| Pod está Running? | kube-state-metrics | ❌ Não | `kube_pod_status_phase{phase="Running"}` |
| Deployment replicas | kube-state-metrics | ❌ Não | `kube_deployment_status_replicas_available` |
| Pod restarts | kube-state-metrics | ❌ Não | `kube_pod_container_status_restarts_total` |
| App respondendo? | Prometheus (up) | ✅ Sim | `up{job="default/myapp"}` |
| Request rate | Aplicação | ✅ Sim | `http_requests_total` |
| Latência P95 | Aplicação | ✅ Sim | `http_request_duration_seconds` |
| Pedidos processados | Aplicação | ✅ Sim | `myapp_orders_total` |
| Fila de background | Aplicação | ✅ Sim | `myapp_queue_size` |

## Arquitetura de Coleta

```
┌──────────────────────────────────────────────────────┐
│                    GKE Node                           │
│                                                        │
│  ┌──────────────┐              ┌──────────────┐      │
│  │   Pod App    │              │   kubelet    │      │
│  │              │              │  (cAdvisor)  │      │
│  │  /metrics ◄──┼──scrape──────┼──────────────┤      │
│  │  (custom)    │              │              │      │
│  └──────────────┘              │  /metrics/   │      │
│        │                       │  cadvisor ◄──┼──┐   │
│        │                       │  (auto)      │  │   │
│        │                       └──────────────┘  │   │
│        │                                         │   │
│        │                       ┌──────────────┐  │   │
│        │                       │     GMP      │  │   │
│        │                       │  Collector   │  │   │
│        └───────scrape──────────►  (DaemonSet) ◄─┘   │
│              (via PodMonitoring)└──────┬───────┘     │
└──────────────────────────────────────────┼───────────┘
                                           │
                       ┌───────────────────┼────────────┐
                       │                   │            │
                       │  ┌────────────────▼──────┐    │
                       │  │  kube-state-metrics  │    │
                       │  │  (expõe métricas do  │    │
                       │  │   estado do K8s API) │    │
                       │  └────────────┬──────────┘    │
                       │               │                │
                       │  ┌────────────▼──────┐        │
                       │  │   GMP Backend     │        │
                       │  │   (Managed TSDB)  │        │
                       │  └───────────────────┘        │
                       └────────────────────────────────┘
```

## Estratégia Recomendada

### Para Infraestrutura (SRE/Platform)

Use métricas **automáticas** (kubelet + kube-state-metrics):

```yaml
# Não precisa de PodMonitoring!
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: infrastructure-alerts
  namespace: default
spec:
  groups:
  - name: infra
    interval: 30s
    rules:
    # Memória alta
    - alert: HighMemory
      expr: |
        (container_memory_working_set_bytes{container!=""}
        / container_spec_memory_limit_bytes{container!=""}) > 0.85
      for: 5m

    # Pod não está Ready
    - alert: PodNotReady
      expr: kube_pod_status_ready{condition="false"} == 1
      for: 5m

    # Deployment degradado
    - alert: DeploymentDegraded
      expr: |
        kube_deployment_status_replicas_available
        <
        kube_deployment_spec_replicas
      for: 10m
```

**Vantagens:**
- ✅ Funciona pra **qualquer app** (mesmo sem instrumentação)
- ✅ Zero código necessário
- ✅ Cobre os alertas mais críticos

### Para Aplicação (Developers)

Instrumente a app com métricas de negócio:

```yaml
# Precisa de PodMonitoring
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: myapp-monitoring
  namespace: production
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s

---
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: application-alerts
  namespace: production
spec:
  groups:
  - name: app
    interval: 30s
    rules:
    # Taxa de erro > 1%
    - alert: HighErrorRate
      expr: |
        rate(http_requests_total{status=~"5.."}[5m])
        /
        rate(http_requests_total[5m])
        > 0.01
      for: 2m

    # Latência P95 > 500ms
    - alert: HighLatency
      expr: |
        histogram_quantile(0.95,
          rate(http_request_duration_seconds_bucket[5m])
        ) > 0.5
      for: 5m

    # Fila crescendo
    - alert: QueueGrowing
      expr: |
        deriv(myapp_queue_size[10m]) > 0
      for: 15m
```

**Vantagens:**
- ✅ Detecta problemas **antes** de afetar infraestrutura
- ✅ Métricas alinhadas com SLOs/SLIs
- ✅ Visibilidade de negócio

## Resumo

**Para monitorar infraestrutura básica:**
→ Use métricas do kubelet + kube-state-metrics
→ Não precisa de PodMonitoring
→ Não precisa instrumentar app

**Para monitorar aplicação:**
→ Instrumente com Prometheus client
→ Exponha `/metrics`
→ Crie PodMonitoring
→ Métricas de negócio + latência + erros

**Para detectar se app está respondendo:**
→ Use PodMonitoring (gera métrica `up`)
→ Alerta quando `sum(up{...}) < 1`

## Referências

- [cAdvisor metrics](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md)
- [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics/tree/main/docs)
- [Prometheus client libraries](https://prometheus.io/docs/instrumenting/clientlibs/)
