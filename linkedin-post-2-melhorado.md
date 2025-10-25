# Post LinkedIn - Google Managed Prometheus na Prática

---

**Google Managed Prometheus: Observabilidade no GKE sem gerenciar infraestrutura** 🚀

Acabei de implementar um sistema completo de alertas no GKE e quero compartilhar o que aprendi na prática sobre o Google Managed Prometheus (GMP).

## 🎯 O que é GMP?

GMP é uma implementação gerenciada do Prometheus que resolve o maior desafio da observabilidade: **não precisar gerenciar a infraestrutura de monitoramento**.

A arquitetura é genial:
- Os **componentes de coleta** rodam no seu cluster
- O **backend** (storage, query engine, HA) é gerenciado pelo Google
- Você só se preocupa com métricas e alertas

## 🏗️ Arquitetura: 4 componentes principais

O GMP instala tudo no namespace `gmp-system`:

**1️⃣ Collector (DaemonSet)**
- Roda em **cada node** do cluster
- Lê os CRDs `PodMonitoring` que você cria
- Faz scrape dos endpoints `/metrics` dos seus pods
- Envia dados pro backend gerenciado (não fica no cluster!)

**2️⃣ Rule Evaluator (Deployment)**
- Lê os CRDs `Rules` que você cria
- Executa queries PromQL contra o backend a cada intervalo (ex: 30s)
- Marca alertas: `PENDING` → `FIRING` → `RESOLVED`
- Manda alertas pro AlertManager

**3️⃣ AlertManager (StatefulSet)**
- Recebe alertas do Rule Evaluator
- Agrupa alertas similares (evita spam)
- Implementa silences e repeat intervals
- Roteia pra receivers: Email, Slack, PagerDuty
- Expõe **API REST** na porta 9093 (essencial pra debug!)

**4️⃣ GMP Operator (Deployment)**
- Gerencia ciclo de vida dos CRDs
- Reconcilia estado desejado vs atual
- Orquestra os outros componentes

## 📝 Na prática: 3 CRDs que você usa

**PodMonitoring** - Define O QUE coletar
```yaml
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: sample-app-monitoring
  namespace: default
spec:
  selector:
    matchLabels:
      app: sample-app
  endpoints:
  - port: metrics        # Nome da porta (não número!)
    interval: 30s        # Scrape a cada 30s
    path: /metrics
```

**Rules** - Define QUANDO alertar
```yaml
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: sample-app-alerts
  namespace: default
spec:
  groups:
  - name: availability
    interval: 30s
    rules:
    - alert: PodDown
      expr: sum(up{job="default/sample-app-monitoring"}) < 1
      for: 1m              # Aguarda 1min antes de disparar
      labels:
        severity: critical
      annotations:
        summary: "App está down!"
```

**OperatorConfig** - Configuração global (SMTP, receivers)
```yaml
apiVersion: monitoring.googleapis.com/v1
kind: OperatorConfig
metadata:
  namespace: gmp-public    # TEM que ser gmp-public!
  name: config
spec:
  managedAlertmanager:
    enabled: true
    configSecret:
      name: alertmanager-config    # Secret com config SMTP
```

## 🧪 Testei na prática: Simulando um crash

Implementei um cenário real no lab: derrubar o app e ver o alerta chegar no email.

**Setup:**
- App: `sample-app` com 2 réplicas
- PodMonitoring configurado (scrape a cada 30s)
- Rule: alerta se `sum(up{...}) < 1` por mais de 1 minuto
- AlertManager: enviando email via Gmail SMTP

**Teste executado:**
```bash
# T+0s: Derrubo todas as réplicas
kubectl scale deployment sample-app --replicas=0

# Acompanho o que acontece...
kubectl get pods -w
```

## ⏱️ Timeline completa (observado no lab):

```
T+0s    kubectl scale --replicas=0
        └─ Pods começam a terminar (Terminating)

T+30s   Collector faz próximo scrape
        └─ Detecta up=0 (targets down)
        └─ Envia pro backend GMP

T+60s   Rule Evaluator executa query
        └─ sum(up{...}) < 1 → true
        └─ Alerta está true há 1min (for: 1m)
        └─ Muda estado: PENDING → FIRING

T+70s   AlertManager recebe o alerta FIRING
        └─ Aguarda group_wait (10s padrão)
        └─ Agrupa alertas similares
        └─ Envia via SMTP pro Gmail

T+75s   📧 Email chega na caixa de entrada!
        Subject: [FIRING:1] PodDown default/sample-app
```

**Total: ~75 segundos** do crash até a notificação.

## 🔍 Validação: API do AlertManager

A API foi crucial pra entender o que estava acontecendo:

```bash
# Port-forward pro AlertManager
kubectl port-forward -n gmp-system svc/alertmanager 9093:9093

# Ver alertas ativos
curl http://localhost:9093/api/v2/alerts | jq
```

**Resposta (alerta FIRING):**
```json
[
  {
    "labels": {
      "alertname": "PodDown",
      "severity": "critical",
      "namespace": "default"
    },
    "status": {
      "state": "active",
      "silencedBy": [],
      "inhibitedBy": []
    },
    "startsAt": "2025-10-25T20:05:00Z",
    "endsAt": "0001-01-01T00:00:00Z",
    "fingerprint": "a1b2c3d4e5f6"
  }
]
```

Isso me mostrou:
✅ O alerta estava ativo (`state: active`)
✅ Não estava silenciado (`silencedBy: []`)
✅ O timestamp exato do disparo

**Depois voltei os pods:**
```bash
kubectl scale deployment sample-app --replicas=2
```

~2 minutos depois, o alerta mudou pra `RESOLVED` e recebi email de resolução!

## 💡 Descobertas importantes

**1) Você NÃO precisa instrumentar pra alertas de infra**

80% dos alertas usam métricas que **já existem**:
- CPU/Memória → kubelet (cAdvisor)
- Status de pods → kube-state-metrics
- A métrica `up` (target health) → GMP

Só precisa instrumentar para:
- Request rate, latência, erros
- Métricas de negócio (vendas, filas)

**2) A API do AlertManager é essencial**

Quando o alerta não chega, use a API pra debugar:
- `/api/v2/alerts` → Ver alertas (FIRING, PENDING, RESOLVED)
- `/api/v2/status` → Ver config aplicada (SMTP, receivers)
- `/api/v2/silences` → Ver silences ativos

**3) Backend gerenciado = menos preocupação**

O que o Google gerencia:
✅ Storage distribuído (24 meses retenção)
✅ Query engine escalável
✅ Alta disponibilidade
✅ Backup automático

O que você gerencia:
📝 CRDs (PodMonitoring, Rules, OperatorConfig)
📝 Secret com SMTP

## 💰 Custos observados

- GMP: **Grátis** até 50GB/mês
- No meu lab: usando ~2-3GB/mês
- Custo real: cluster GKE Autopilot (~$50-75/mês)

## 🎁 Repositório com tudo documentado

Coloquei tudo no GitHub com:
- Setup completo passo a passo (~15min)
- Todos os YAMLs prontos pra usar
- Como configurar SMTP (Gmail + App Password)
- Troubleshooting e comandos de debug
- Arquitetura detalhada com diagramas
- Este mesmo teste que executei no lab

Link nos comentários! 👇

---

**O que você achou dessa abordagem de observabilidade gerenciada?**

Já usou GMP, Amazon Managed Prometheus ou Grafana Cloud? Compartilha a experiência!

#Kubernetes #GCP #Observability #SRE #DevOps #Prometheus #CloudNative #GKE

---

## 📸 Imagens sugeridas:

1️⃣ Screenshot: 4 pods do GMP rodando em `gmp-system`
```bash
kubectl get pods -n gmp-system
```

2️⃣ Screenshot: App escalado pra 0 réplicas
```bash
kubectl get pods  # Mostrando 0/0 ou Terminating
```

3️⃣ Screenshot: JSON do alerta na API
```bash
curl localhost:9093/api/v2/alerts | jq
```

4️⃣ Screenshot: Email recebido com `[FIRING:1] PodDown`

---

## 💬 Comentário com link do repo:

```
📚 Repositório completo com toda a implementação:
[seu-link-github]

Inclui:
✅ Setup passo a passo
✅ YAMLs prontos pra copiar
✅ Troubleshooting detalhado
✅ Comandos de debug
✅ Arquitetura explicada

Clone e teste você mesmo em 15 minutos! 🚀
```
