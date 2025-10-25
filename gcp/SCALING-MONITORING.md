# Escalando o Monitoramento no GMP

## TL;DR

**Novo node?** → ✅ Automático (Collector é DaemonSet)
**Novo pod com mesmas labels?** → ✅ Automático (PodMonitoring já cobre)
**Novo deployment com labels diferentes?** → ❌ Manual (precisa criar/atualizar PodMonitoring)

## Cenários de Escalabilidade

### 1. Adicionando Nodes ao Cluster

**O que acontece:**
✅ **Automático!** O Collector é um DaemonSet, então automaticamente roda em cada novo node.

**Exemplo:**
```bash
# Cluster com 2 nodes
kubectl get pods -n gmp-system -l app.kubernetes.io/name=collector
# collector-abc12  (node-1)
# collector-def34  (node-2)

# Adiciona 1 node novo
gcloud container clusters resize meu-cluster --num-nodes=3

# GMP automaticamente cria novo Collector
kubectl get pods -n gmp-system -l app.kubernetes.io/name=collector
# collector-abc12  (node-1)
# collector-def34  (node-2)
# collector-ghi56  (node-3)  ← NOVO
```

**Métricas afetadas:**
- ✅ Métricas do kubelet (CPU/mem) → coletadas automaticamente do novo node
- ✅ Métricas de pods que rodarem no novo node → scraped automaticamente

**Você não precisa fazer NADA!**

---

### 2. Escalando Deployment Existente (Mais Réplicas)

**O que acontece:**
✅ **Automático!** Se o deployment já é monitorado, novos pods são scraped automaticamente.

**Exemplo:**
```yaml
# PodMonitoring existente
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: sample-app-monitoring
  namespace: default
spec:
  selector:
    matchLabels:
      app: sample-app  # ← Monitora TODOS os pods com essa label
  endpoints:
  - port: metrics
    interval: 30s
```

```bash
# Escala de 2 para 5 réplicas
kubectl scale deployment sample-app --replicas=5

# Collector automaticamente detecta e scrape os 5 pods
kubectl get pods -l app=sample-app
# sample-app-79c6ddcd8c-abc12
# sample-app-79c6ddcd8c-def34
# sample-app-79c6ddcd8c-ghi56  ← NOVO
# sample-app-79c6ddcd8c-jkl78  ← NOVO
# sample-app-79c6ddcd8c-mno90  ← NOVO
```

**Métricas:**
A métrica `up{job="default/sample-app-monitoring"}` agora terá 5 séries temporais (uma por pod).

**Query de alerta continua funcionando:**
```yaml
expr: sum(up{job="default/sample-app-monitoring"}) < 1
# Antes: sum = 2
# Agora: sum = 5
# Alerta só dispara se TODOS os 5 pods caírem
```

**Você não precisa fazer NADA!**

---

### 3. Novo Deployment com as MESMAS Labels

**O que acontece:**
✅ **Automático!** O PodMonitoring usa label selector, então qualquer pod com aquelas labels é monitorado.

**Exemplo:**
```yaml
# PodMonitoring existente (genérico)
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: all-apps-monitoring
  namespace: production
spec:
  selector:
    matchLabels:
      monitoring: enabled  # ← Label genérica
  endpoints:
  - port: metrics
    interval: 30s
```

```yaml
# Novo deployment usando a mesma label
apiVersion: apps/v1
kind: Deployment
metadata:
  name: new-api
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: new-api
      monitoring: enabled  # ← Mesma label do PodMonitoring!
  template:
    metadata:
      labels:
        app: new-api
        monitoring: enabled
    spec:
      containers:
      - name: api
        image: my-api:v1
        ports:
        - containerPort: 8080
          name: metrics  # ← Nome tem que bater
```

**Resultado:**
✅ Collector automaticamente scrape o `new-api` deployment
✅ Métricas aparecem com labels: `{job="production/all-apps-monitoring", pod="new-api-..."}`

**Você não precisa fazer NADA!**

---

### 4. Novo Deployment com Labels DIFERENTES

**O que acontece:**
❌ **Manual!** Você precisa criar um novo PodMonitoring OU atualizar o existente.

**Opção A: Criar novo PodMonitoring (recomendado)**

```yaml
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: frontend-monitoring
  namespace: production
spec:
  selector:
    matchLabels:
      app: frontend  # ← Label específica
  endpoints:
  - port: http  # ← Nome diferente de porta
    interval: 30s
    path: /metrics
```

```bash
kubectl apply -f frontend-monitoring.yaml
```

**Vantagens:**
- ✅ Job names separados: `production/frontend-monitoring`
- ✅ Pode ter configurações diferentes (interval, path)
- ✅ Alertas podem ter severidades diferentes

**Opção B: Atualizar PodMonitoring existente (label genérica)**

```yaml
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: all-apps-monitoring
  namespace: production
spec:
  selector:
    matchLabels:
      tier: backend  # ← Label mais genérica
  endpoints:
  - port: metrics
    interval: 30s
```

Todos os deployments precisam ter label `tier: backend`:
```yaml
labels:
  app: api
  tier: backend
---
labels:
  app: worker
  tier: backend
---
labels:
  app: frontend
  tier: backend
```

**Desvantagem:**
- ❌ Todos os pods ficam no mesmo job name
- ❌ Mais difícil de criar alertas específicos por app

---

### 5. Novo Namespace

**O que acontece:**
❌ **Manual!** PodMonitoring é namespaced, então cada namespace precisa do seu.

**Opção A: PodMonitoring por namespace**

```yaml
# namespace: production
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: apps-monitoring
  namespace: production
spec:
  selector:
    matchLabels:
      monitoring: enabled
  endpoints:
  - port: metrics
    interval: 30s
---
# namespace: staging
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: apps-monitoring
  namespace: staging
spec:
  selector:
    matchLabels:
      monitoring: enabled
  endpoints:
  - port: metrics
    interval: 30s
```

**Job names gerados:**
- `production/apps-monitoring`
- `staging/apps-monitoring`

**Opção B: ClusterPodMonitoring (todos os namespaces)**

```yaml
apiVersion: monitoring.googleapis.com/v1
kind: ClusterPodMonitoring
metadata:
  name: all-apps-monitoring
spec:
  selector:
    matchLabels:
      monitoring: enabled
  endpoints:
  - port: metrics
    interval: 30s
```

**Vantagem:**
✅ Monitora pods em TODOS os namespaces
✅ Um único YAML

**Desvantagem:**
❌ Menos granular (mesmas configs pra tudo)
❌ Job name não inclui namespace automaticamente

---

## Estratégias de Monitoramento

### Estratégia 1: Um PodMonitoring por Deployment (Mais Controle)

**Estrutura:**
```
k8s-manifests/
├── api/
│   ├── deployment.yaml
│   └── monitoring.yaml  ← PodMonitoring específico
├── worker/
│   ├── deployment.yaml
│   └── monitoring.yaml  ← PodMonitoring específico
└── frontend/
    ├── deployment.yaml
    └── monitoring.yaml  ← PodMonitoring específico
```

**Exemplo:**
```yaml
# api/monitoring.yaml
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: api-monitoring
  namespace: production
spec:
  selector:
    matchLabels:
      app: api
  endpoints:
  - port: metrics
    interval: 15s  # ← API precisa scrape mais frequente
```

```yaml
# worker/monitoring.yaml
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: worker-monitoring
  namespace: production
spec:
  selector:
    matchLabels:
      app: worker
  endpoints:
  - port: metrics
    interval: 60s  # ← Worker pode ser menos frequente
```

**Vantagens:**
- ✅ Configurações customizadas por app
- ✅ Job names claros: `production/api-monitoring`, `production/worker-monitoring`
- ✅ Alertas específicos por app

**Desvantagens:**
- ❌ Mais YAMLs pra gerenciar
- ❌ Precisa criar PodMonitoring pra cada novo deployment

---

### Estratégia 2: Um PodMonitoring por Namespace (Mais Simples)

**Estrutura:**
```yaml
# production-monitoring.yaml
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: apps-monitoring
  namespace: production
spec:
  selector:
    matchLabels:
      monitoring: enabled  # ← Label padrão
  endpoints:
  - port: metrics
    interval: 30s
```

Todos os deployments usam a mesma label:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: production
spec:
  template:
    metadata:
      labels:
        app: api
        monitoring: enabled  # ← Adiciona essa label
```

**Vantagens:**
- ✅ Simples: um YAML por namespace
- ✅ Novos deployments são monitorados automaticamente (se tiverem a label)

**Desvantagens:**
- ❌ Job name único: `production/apps-monitoring` (todas as apps juntas)
- ❌ Mesmas configs pra todos (interval, path)

---

### Estratégia 3: ClusterPodMonitoring (Cluster-Wide)

**Estrutura:**
```yaml
# cluster-monitoring.yaml
apiVersion: monitoring.googleapis.com/v1
kind: ClusterPodMonitoring
metadata:
  name: all-apps
spec:
  selector:
    matchLabels:
      monitoring: enabled
  endpoints:
  - port: metrics
    interval: 30s
```

**Vantagens:**
- ✅ Um único YAML pra todo o cluster
- ✅ Funciona em qualquer namespace

**Desvantagens:**
- ❌ Menos flexibilidade (mesma config pra tudo)
- ❌ Job names menos claros

---

## Checklist: Adicionando Novo Deployment

### Se o deployment expõe `/metrics`:

1. **Define a label de monitoramento**
   ```yaml
   labels:
     app: myapp
     monitoring: enabled  # ← Adiciona essa
   ```

2. **Garante que a porta tem NOME**
   ```yaml
   ports:
   - containerPort: 8080
     name: metrics  # ← NOME, não número!
   ```

3. **Escolhe estratégia:**

   **Opção A: Usar PodMonitoring existente**
   - Se já existe um com label `monitoring: enabled` → pronto!
   - Deploy automaticamente monitorado

   **Opção B: Criar novo PodMonitoring**
   ```yaml
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
   ```

4. **Aplica**
   ```bash
   kubectl apply -f myapp-deployment.yaml
   kubectl apply -f myapp-monitoring.yaml  # Se criou novo
   ```

5. **Verifica**
   ```bash
   # Checa se PodMonitoring foi criado
   kubectl get podmonitoring -n production

   # Checa se pods têm as labels corretas
   kubectl get pods -n production -l app=myapp --show-labels

   # Aguarda 1-2 minutos e verifica métricas
   # (Via Metrics Explorer ou query)
   ```

### Se o deployment NÃO expõe `/metrics`:

**IMPORTANTE:** Se seu app não tem endpoint `/metrics`, você:
- ❌ **NÃO** precisa criar `PodMonitoring`
- ✅ **SIM** pode criar `Rules` para alertas de infraestrutura

**Por quê?** Métricas de CPU, memória e status dos pods **já existem automaticamente** via kubelet e kube-state-metrics. O GMP coleta essas métricas sem você fazer nada.

**Exemplo prático:**

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        # ❌ NÃO tem label "monitoring: enabled"
    spec:
      containers:
      - name: app
        image: myapp:v1.0
        # ❌ NÃO expõe porta /metrics
        ports:
        - containerPort: 8080
          name: http
```

**Nesse caso, você faz:**

```yaml
# ❌ NÃO CRIA PodMonitoring
# (Porque o app não tem /metrics pra fazer scrape)

# ✅ SÓ CRIA Rules (alertas de infraestrutura)
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: myapp-infra-alerts
  namespace: production
spec:
  groups:
  - name: infra
    interval: 30s
    rules:
    # Memória alta (usa métrica do kubelet)
    - alert: MyAppHighMemory
      expr: |
        (container_memory_working_set_bytes{namespace="production", pod=~"myapp-.*", container!=""}
        / container_spec_memory_limit_bytes{namespace="production", pod=~"myapp-.*", container!=""}) > 0.85
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "MyApp usando muita memória"
        description: "Pod {{ $labels.pod }}: {{ $value | humanizePercentage }}"

    # Pod não está Running (usa métrica do kube-state-metrics)
    - alert: MyAppPodNotRunning
      expr: |
        kube_pod_status_phase{namespace="production", pod=~"myapp-.*", phase!="Running", phase!="Succeeded"} == 1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod do MyApp em estado {{ $labels.phase }}"
        description: "Pod {{ $labels.pod }} está em {{ $labels.phase }}"

    # CPU throttling (usa métrica do kubelet)
    - alert: MyAppCPUThrottling
      expr: |
        rate(container_cpu_cfs_throttled_seconds_total{namespace="production", pod=~"myapp-.*"}[5m]) > 0.5
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "MyApp com CPU throttling"
        description: "Pod {{ $labels.pod }} está sendo throttled"

    # Pods reiniciando muito (usa métrica do kube-state-metrics)
    - alert: MyAppCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total{namespace="production", pod=~"myapp-.*"}[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "MyApp reiniciando frequentemente"
        description: "Pod {{ $labels.pod }} crashlooping"
```

**Essas métricas JÁ EXISTEM no GMP!** Não precisa de PodMonitoring.

**Resumo visual:**

```
Cenário 1: App SEM /metrics
┌─────────────────────────────────┐
│  Deployment (sem /metrics)      │
│  - Não expõe porta metrics      │
│  - Não tem instrumentação       │
└─────────────────────────────────┘
         │
         ├─ ❌ NÃO cria PodMonitoring
         │
         └─ ✅ Cria Rules (alertas de infra)
                  ↓
            Usa métricas automáticas:
            - container_memory_* (kubelet)
            - container_cpu_* (kubelet)
            - kube_pod_status_* (kube-state-metrics)

Cenário 2: App COM /metrics
┌─────────────────────────────────┐
│  Deployment (com /metrics)      │
│  - Expõe porta 8080/metrics     │
│  - Tem instrumentação Prometheus│
└─────────────────────────────────┘
         │
         ├─ ✅ Cria PodMonitoring
         │     ↓
         │   Coleta métricas custom:
         │   - http_requests_total
         │   - myapp_orders_processed
         │
         └─ ✅ Cria Rules (alertas de app + infra)
                  ↓
            Usa métricas custom + automáticas:
            - http_requests_total (app)
            - myapp_* (app)
            - container_memory_* (kubelet)
            - kube_pod_status_* (kube-state-metrics)
```

---

## Exemplo Completo: Adicionando Novo Deployment

```yaml
# myapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        monitoring: enabled  # ← 1. Adiciona label de monitoring
    spec:
      containers:
      - name: app
        image: myapp:v1.0
        ports:
        - containerPort: 8080
          name: metrics  # ← 2. Porta com NOME
        - containerPort: 8081
          name: http
---
# myapp-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
spec:
  selector:
    app: myapp
  ports:
  - port: 8080
    name: metrics  # ← 3. Nome bate com o deployment
  - port: 8081
    name: http
---
# myapp-monitoring.yaml (SE não tiver PodMonitoring genérico)
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: myapp-monitoring
  namespace: production
spec:
  selector:
    matchLabels:
      app: myapp  # ← 4. Selector bate com pod labels
  endpoints:
  - port: metrics  # ← 5. Nome da porta (não número!)
    interval: 30s
    path: /metrics
---
# myapp-alerts.yaml
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: myapp-alerts
  namespace: production
spec:
  groups:
  - name: myapp
    interval: 30s
    rules:
    - alert: MyAppDown
      expr: sum(up{job="production/myapp-monitoring"}) < 1
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "MyApp não está respondendo"
        description: "{{ $value }} pods ativos"
```

**Deploy:**
```bash
kubectl apply -f myapp-deployment.yaml
kubectl apply -f myapp-service.yaml
kubectl apply -f myapp-monitoring.yaml
kubectl apply -f myapp-alerts.yaml

# Verifica
kubectl get pods -n production -l app=myapp
kubectl get podmonitoring -n production myapp-monitoring
kubectl get rules -n production myapp-alerts
```

---

## Resumo

| Cenário | Automático? | Ação Necessária |
|---------|-------------|-----------------|
| Adicionar node | ✅ Sim | Nenhuma |
| Escalar deployment existente | ✅ Sim | Nenhuma |
| Novo deployment com mesmas labels | ✅ Sim | Nenhuma |
| Novo deployment com labels diferentes | ❌ Não | Criar/atualizar PodMonitoring |
| Novo namespace | ❌ Não | Criar PodMonitoring no namespace |
| Métricas de infra (CPU/mem) | ✅ Sim | Nenhuma (kubelet coleta) |
| Métricas de app (custom) | ❌ Não | Instrumentar app + PodMonitoring |

### Decisão: Preciso criar PodMonitoring?

**Use essa árvore de decisão:**

```
Seu app expõe /metrics?
│
├─ ❌ NÃO
│   │
│   └─ Não precisa de PodMonitoring!
│      Só cria Rules usando métricas automáticas:
│      - container_memory_* (kubelet)
│      - container_cpu_* (kubelet)
│      - kube_pod_status_* (kube-state-metrics)
│
└─ ✅ SIM
    │
    └─ Já existe PodMonitoring com label selector que match?
        │
        ├─ ✅ SIM
        │   │
        │   └─ Não precisa criar!
        │      O deployment será monitorado automaticamente
        │
        └─ ❌ NÃO
            │
            └─ Precisa criar PodMonitoring
               com selector que faça match com as labels do pod
```

**Regra de ouro:**
- **App SEM /metrics** → ❌ NÃO cria PodMonitoring (usa métricas automáticas do kubelet/kube-state-metrics)
- **App COM /metrics** → ✅ Cria PodMonitoring OU usa um existente com label selector que match
