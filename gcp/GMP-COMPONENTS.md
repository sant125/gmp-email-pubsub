# Componentes do Google Managed Prometheus (GMP)

## Visão Geral

O GMP é a implementação gerenciada do Prometheus pelo Google Cloud. Ele roda dentro do seu cluster GKE mas é gerenciado pelo Google, então você não precisa se preocupar com escalabilidade, storage ou alta disponibilidade.

## Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                        GKE Cluster                               │
│                                                                   │
│  ┌───────────────┐                                               │
│  │  Namespace:   │                                               │
│  │   default     │                                               │
│  │               │                                               │
│  │  ┌─────────┐  │      ┌──────────────────────────────┐        │
│  │  │  Pod    │  │      │   Namespace: gmp-system      │        │
│  │  │ sample  │  │      │                              │        │
│  │  │  -app   │◄─┼──────┼─┐ Collector (DaemonSet)     │        │
│  │  └─────────┘  │      │ │   - Scrape /metrics       │        │
│  │    expõe      │      │ │   - Coleta a cada 30s     │        │
│  │   /metrics    │      │ │   - Envia pro backend     │        │
│  └───────────────┘      │ └───────────────────────────┘        │
│                         │                                        │
│  ┌───────────────┐      │ ┌───────────────────────────┐        │
│  │  Namespace:   │      │ │  Rule Evaluator           │        │
│  │  gmp-public   │      │ │   - Lê Rules (CRDs)       │        │
│  │               │      │ │   - Executa queries PromQL│        │
│  │ PodMonitoring │◄─────┼─┤   - Gera alertas          │        │
│  │ Rules         │      │ └───────────────────────────┘        │
│  │ OperatorConfig│      │                                        │
│  └───────────────┘      │ ┌───────────────────────────┐        │
│                         │ │  AlertManager             │        │
│                         │ │   - Recebe alertas        │        │
│                         │ │   - Agrupa/roteia         │        │
│                         │ │   - Envia SMTP            │        │
│                         │ └───────────────────────────┘        │
│                         │                                        │
│                         │ ┌───────────────────────────┐        │
│                         │ │  GMP Operator             │        │
│                         │ │   - Gerencia CRDs         │        │
│                         │ │   - Reconcilia config     │        │
│                         │ └───────────────────────────┘        │
│                         └──────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────┐
                    │  Google Cloud Managed     │
                    │  Prometheus Backend       │
                    │  - Storage (TSDB)         │
                    │  - Query Engine           │
                    │  - Long-term retention    │
                    └───────────────────────────┘
```

## Componentes Principais

### 1. Collector (DaemonSet)

**O que faz:**
- Roda em **cada node** do cluster (DaemonSet)
- Lê os `PodMonitoring` e `ClusterPodMonitoring` CRDs
- Faz scrape dos endpoints `/metrics` dos pods
- Envia métricas pro backend gerenciado do GMP

**Como funciona:**
```yaml
# Você cria um PodMonitoring
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: sample-app-monitoring
  namespace: default
spec:
  selector:
    matchLabels:
      app: sample-app  # ← Collector procura pods com essa label
  endpoints:
  - port: metrics      # ← Scrape essa porta
    interval: 30s      # ← A cada 30 segundos
    path: /metrics     # ← Nesse path
```

**O Collector:**
1. Detecta o PodMonitoring
2. Procura pods com label `app=sample-app`
3. Conecta na porta `metrics` (8080)
4. Faz GET em `/metrics` a cada 30s
5. Envia dados pro backend do GMP

**Logs úteis:**
```bash
kubectl logs -n gmp-system -l app.kubernetes.io/name=collector
```

### 2. Rule Evaluator (Deployment)

**O que faz:**
- Lê os `Rules` CRDs que você cria
- Executa queries PromQL contra o backend do GMP
- Avalia se alertas devem disparar
- Envia alertas pro AlertManager

**Como funciona:**
```yaml
# Você cria uma Rule
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: lab-alerts
  namespace: default
spec:
  groups:
  - name: lab_alerts
    interval: 30s  # ← Rule Evaluator executa query a cada 30s
    rules:
    - alert: PodDown
      expr: sum(up{job="default/sample-app-monitoring"}) < 1
      for: 1m  # ← Precisa estar true por 1 minuto antes de disparar
```

**O Rule Evaluator:**
1. Lê a Rule a cada 30s (interval)
2. Executa a query PromQL no backend do GMP
3. Se resultado = true, marca alerta como PENDING
4. Aguarda 1 minuto (`for: 1m`)
5. Se ainda true, muda pra FIRING
6. Envia pro AlertManager

**Logs úteis:**
```bash
kubectl logs -n gmp-system -l app.kubernetes.io/name=rule-evaluator -c evaluator
```

### 3. AlertManager (StatefulSet)

**O que faz:**
- Recebe alertas do Rule Evaluator
- Agrupa alertas similares
- Roteia pra receivers (email, Slack, PagerDuty)
- Implementa silences e inibição

**Como funciona:**
```yaml
# Você cria um Secret com config SMTP
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-config
  namespace: gmp-public  # ← TEM que ser gmp-public
stringData:
  alertmanager.yaml: |
    route:
      receiver: 'email-alerts'
      group_wait: 10s       # ← Aguarda 10s pra agrupar alertas
      group_interval: 5m    # ← Envia grupo novo a cada 5min
      repeat_interval: 3h   # ← Re-envia se ainda ativo após 3h

    receivers:
    - name: 'email-alerts'
      email_configs:
      - to: 'seu@email.com'
        send_resolved: true  # ← Envia email quando resolver também
```

**O AlertManager:**
1. Recebe alerta FIRING do Rule Evaluator
2. Aguarda 10s (`group_wait`) pra agrupar outros alertas similares
3. Envia email pro receiver configurado
4. Se alerta continuar ativo, re-envia após 3h (`repeat_interval`)
5. Quando alerta resolver, envia email de RESOLVED (se `send_resolved: true`)

**API REST (útil pra debug):**
```bash
kubectl port-forward -n gmp-system svc/alertmanager 9093:9093

# Ver alertas ativos
curl http://localhost:9093/api/v2/alerts

# Ver status
curl http://localhost:9093/api/v2/status
```

### 4. GMP Operator (Deployment)

**O que faz:**
- Gerencia o ciclo de vida dos CRDs (PodMonitoring, Rules, OperatorConfig)
- Monitora mudanças e reconcilia estado desejado
- Configura Collector, Rule Evaluator e AlertManager

**Como funciona:**
```yaml
# Você cria OperatorConfig
apiVersion: monitoring.googleapis.com/v1
kind: OperatorConfig
metadata:
  namespace: gmp-public
  name: config
spec:
  managedAlertmanager:
    enabled: true
    configSecret:
      name: alertmanager-config  # ← Operator aplica essa config
```

**O Operator:**
1. Detecta mudança no OperatorConfig
2. Lê o Secret `alertmanager-config`
3. Aplica config no AlertManager
4. Reinicia pods se necessário

## CRDs (Custom Resource Definitions)

### PodMonitoring

**Propósito:** Define quais pods devem ter métricas coletadas

**Campos importantes:**
- `selector.matchLabels`: Quais pods monitorar
- `endpoints[].port`: Nome da porta (não número!)
- `endpoints[].path`: Path das métricas (geralmente `/metrics`)
- `endpoints[].interval`: Frequência de coleta

**Exemplo:**
```yaml
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: sample-app-monitoring
  namespace: default  # ← Pode ser qualquer namespace
spec:
  selector:
    matchLabels:
      app: sample-app
  endpoints:
  - port: metrics  # ← Nome, não número! Vem do Service
    interval: 30s
    path: /metrics
```

**Job name gerado:**
O GMP cria automaticamente o job name no formato:
```
{namespace}/{podmonitoring-name}
```
Exemplo: `default/sample-app-monitoring`

### Rules

**Propósito:** Define regras de alerta (quando disparar, pra quem enviar)

**Campos importantes:**
- `spec.groups[].interval`: Frequência de avaliação da query
- `rules[].alert`: Nome do alerta
- `rules[].expr`: Query PromQL
- `rules[].for`: Tempo que condição deve ser true antes de disparar
- `rules[].labels.severity`: Severidade (critical, warning, info)

**Exemplo:**
```yaml
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: lab-alerts
  namespace: default  # ← Pode ser qualquer namespace
spec:
  groups:
  - name: lab_alerts
    interval: 30s
    rules:
    - alert: PodDown
      expr: sum(up{job="default/sample-app-monitoring"}) < 1
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Pod não está respondendo"
        description: "Sample-app: {{ $value }} pods respondendo"
```

**Status:**
Você pode verificar se a Rule foi aceita:
```bash
kubectl get rules lab-alerts -n default -o yaml | grep -A 5 status

# Deve mostrar:
# status:
#   conditions:
#   - status: "True"
#     type: ConfigurationCreateSuccess
```

### OperatorConfig

**Propósito:** Configuração global do GMP (habilita features, aponta pro Secret do AlertManager)

**Campos importantes:**
- `managedAlertmanager.enabled`: Habilita AlertManager gerenciado
- `managedAlertmanager.configSecret`: Aponta pro Secret com config SMTP
- `features.targetStatus.enabled`: Mostra status dos targets sendo scraped

**Exemplo:**
```yaml
apiVersion: monitoring.googleapis.com/v1
kind: OperatorConfig
metadata:
  namespace: gmp-public  # ← TEM que ser gmp-public
  name: config           # ← TEM que ser "config"
spec:
  features:
    targetStatus:
      enabled: true
  managedAlertmanager:
    enabled: true
    configSecret:
      name: alertmanager-config
      key: alertmanager.yaml
```

**Importante:**
- O OperatorConfig **sempre** fica em `gmp-public`
- O Secret do AlertManager **também** tem que estar em `gmp-public`

## Namespaces no GMP

### gmp-system
- **Gerenciado pelo Google**
- Contém os pods: Collector, Rule Evaluator, AlertManager, Operator
- Você **não cria recursos aqui**, só lê logs

### gmp-public
- **Gerenciado por você**
- Contém: OperatorConfig, Secret do AlertManager
- É onde você configura o comportamento global do GMP

### default (ou qualquer outro)
- **Gerenciado por você**
- Contém: PodMonitoring, Rules, seus apps
- É onde você define o que monitorar e os alertas

## Timeline de um Alerta

Exemplo: Pod cai às 10:00:00

```
10:00:00  Pod morre
          └─> up{job="default/sample-app-monitoring"} = 0

10:00:30  Collector faz scrape
          └─> Detecta up=0, envia pro backend GMP

10:01:00  Rule Evaluator executa query (interval: 30s)
          └─> sum(up{...}) < 1 = true
          └─> Marca alerta como PENDING

10:02:00  Rule Evaluator executa query novamente (interval: 30s)
          └─> Ainda true, passou 1min (for: 1m)
          └─> Alerta vira FIRING
          └─> Envia pro AlertManager

10:02:10  AlertManager aguarda group_wait (10s)
          └─> Agrupa alertas similares

10:02:10  AlertManager envia email
          └─> SMTP → Gmail

10:02:15  Email chega na caixa de entrada
```

**Total:** ~2-3 minutos do pod cair até email chegar

## Fluxo de Dados

```
1. App expõe /metrics
   ↓
2. Collector faz scrape a cada 30s
   ↓
3. Metrics vão pro GMP Backend (TSDB gerenciado)
   ↓
4. Rule Evaluator executa queries PromQL a cada 30s
   ↓
5. Se alerta disparar, envia pro AlertManager
   ↓
6. AlertManager agrupa e roteia
   ↓
7. Email/Slack/PagerDuty recebe notificação
```

## Diferença: GMP vs Prometheus Self-Hosted

| Feature | GMP | Prometheus Self-Hosted |
|---------|-----|------------------------|
| Storage | ✅ Gerenciado pelo Google | ❌ Você precisa provisionar (PV) |
| Escalabilidade | ✅ Automática | ❌ Você precisa configurar sharding |
| Alta disponibilidade | ✅ Incluída | ❌ Você precisa configurar réplicas |
| Backup | ✅ Automático | ❌ Sua responsabilidade |
| Custo | 💰 Paga por ingestão | 🆓 Grátis (mas paga infra) |
| AlertManager | ✅ Incluído e gerenciado | ❌ Deploy separado |
| Retenção | ✅ 24 meses default | ⚙️  Configurável (limitado por disco) |
| Query | 🌐 Via Cloud Monitoring | 🖥️  UI local do Prometheus |

## Troubleshooting

### Métricas não aparecem

**Checklist:**
1. Collector rodando? `kubectl get pods -n gmp-system`
2. PodMonitoring criado? `kubectl get podmonitoring -A`
3. Pods têm a label correta? `kubectl get pods --show-labels`
4. App expõe `/metrics`? `kubectl exec pod -- wget -qO- localhost:8080/metrics`
5. Porta tem **nome** no Service? `port: 8080, name: metrics`

**Logs do Collector:**
```bash
kubectl logs -n gmp-system -l app.kubernetes.io/name=collector --tail=100
```

### Alertas não disparam

**Checklist:**
1. Rule criada? `kubectl get rules -A`
2. Status = ConfigurationCreateSuccess? `kubectl describe rules nome-da-rule`
3. Query PromQL está correta? Testa no Metrics Explorer
4. Namespace correto no job name? Usa `{namespace}/`{podmonitoring-name}`

**Logs do Rule Evaluator:**
```bash
kubectl logs -n gmp-system -l app.kubernetes.io/name=rule-evaluator -c evaluator
```

### Email não chega

**Checklist:**
1. Secret criado em `gmp-public`? `kubectl get secret -n gmp-public`
2. OperatorConfig aponta pro Secret? `kubectl get operatorconfig -n gmp-public -o yaml`
3. SMTP config correta? Senha = App Password do Gmail
4. Checa spam!

**Logs do AlertManager:**
```bash
kubectl logs -n gmp-system alertmanager-0 -c alertmanager
```

## Referências

- [GMP Docs](https://cloud.google.com/stackdriver/docs/managed-prometheus)
- [PodMonitoring API](https://cloud.google.com/stackdriver/docs/managed-prometheus/setup-managed)
- [Rules API](https://cloud.google.com/stackdriver/docs/managed-prometheus/alerting)
