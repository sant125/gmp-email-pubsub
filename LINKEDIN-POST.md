# Post LinkedIn - Observabilidade com Google Managed Prometheus

## Versão 1: Técnica e Detalhada (post longo)

---

**Implementando Observabilidade em Kubernetes com Google Managed Prometheus (GMP)**

Recentemente implementei um sistema completo de alertas no GKE usando Google Managed Prometheus, e descobri várias armadilhas que podem economizar horas de debug pra quem está começando. 🧵

**O Problema**
Gerenciar Prometheus self-hosted em produção é trabalhoso: precisa configurar storage persistente, pensar em escalabilidade, backup, alta disponibilidade. E se o Prometheus cai, você fica cego.

**A Solução: GMP**
O Google Managed Prometheus é uma implementação gerenciada que roda no seu cluster, mas toda a complexidade (storage, scaling, HA) é abstraída. Você só se preocupa com o que importa: métricas e alertas.

**Arquitetura do GMP**

O GMP é composto por 4 componentes principais rodando no namespace `gmp-system`:

1️⃣ **Collector** (DaemonSet)
- Roda em cada node do cluster
- Lê os CRDs PodMonitoring que você cria
- Faz scrape dos endpoints /metrics dos pods
- Envia dados pro backend gerenciado do GMP

2️⃣ **Rule Evaluator** (Deployment)
- Lê os CRDs Rules que você cria
- Executa queries PromQL contra o backend
- Avalia se alertas devem disparar
- Envia alertas pro AlertManager

3️⃣ **AlertManager** (StatefulSet)
- Recebe alertas do Rule Evaluator
- Agrupa alertas similares
- Roteia para receivers (Email, Slack, PagerDuty)
- Implementa silences e repeat intervals

4️⃣ **GMP Operator** (Deployment)
- Gerencia o ciclo de vida dos CRDs
- Reconcilia estado desejado vs atual
- Configura os componentes acima

**Descoberta #1: Você NÃO precisa instrumentar sua app para alertas básicos**

Essa foi uma revelação importante. Existem 3 fontes de métricas no K8s:

📊 **Métricas do kubelet (cAdvisor)**
- CPU, memória, disco, rede
- Coletadas automaticamente via cgroups
- Já disponíveis no GMP sem fazer nada
- Exemplos: `container_memory_working_set_bytes`, `container_cpu_usage_seconds_total`

📊 **Métricas do kube-state-metrics**
- Estado dos recursos K8s (Pods, Deployments, Nodes)
- Também coletadas automaticamente
- Exemplos: `kube_pod_status_phase`, `kube_deployment_status_replicas_available`

📊 **Métricas da aplicação (custom)**
- Request rate, latência, erros (Golden Signals)
- Métricas de negócio (vendas, filas, processamentos)
- Requer instrumentação (Prometheus client library)
- Precisa expor `/metrics` e criar `PodMonitoring` CRD

**Exemplo prático:**

Para alertar sobre memória alta, você NÃO precisa instrumentar nada:

```yaml
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: infra-alerts
spec:
  groups:
  - name: resources
    rules:
    - alert: HighMemoryUsage
      expr: |
        (container_memory_working_set_bytes{container!=""}
        / container_spec_memory_limit_bytes{container!=""}) > 0.85
      for: 5m
```

Essa métrica já existe! O kubelet coleta automaticamente.

Mas para alertar sobre alta taxa de erro HTTP, você precisa instrumentar:

```yaml
- alert: HighErrorRate
  expr: |
    rate(http_requests_total{status=~"5.."}[5m])
    /
    rate(http_requests_total[5m]) > 0.01
  for: 2m
```

Essa métrica `http_requests_total` só existe se sua app expor.

**Descoberta #2: NUNCA use `or absent()` em alertas**

Esse foi um bug que me custou horas. A query que parecia correta:

```yaml
# ❌ ERRADO
expr: up{job="default/myapp"} == 0 or absent(up{job="default/myapp"})
```

O problema: o alerta dispara quando os pods caem, mas NUNCA resolve quando voltam!

Por quê? O `absent()` pode retornar `true` mesmo com pods ativos se houver séries temporais antigas no TSDB.

**Solução correta:**

```yaml
# ✅ CORRETO
expr: sum(up{job="default/myapp"}) < 1
```

Simples, confiável, e resolve automaticamente quando os pods voltam.

**CRDs: A Interface com o GMP**

O GMP usa Custom Resource Definitions para configuração:

**PodMonitoring**: Define o que coletar
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
  - port: metrics  # Nome da porta (não número!)
    interval: 30s
    path: /metrics
```

**Rules**: Define quando alertar
```yaml
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: myapp-alerts
  namespace: default
spec:
  groups:
  - name: app_alerts
    interval: 30s
    rules:
    - alert: PodDown
      expr: sum(up{job="default/myapp-monitoring"}) < 1
      for: 1m
      labels:
        severity: critical
```

**OperatorConfig**: Configuração global (em gmp-public!)
```yaml
apiVersion: monitoring.googleapis.com/v1
kind: OperatorConfig
metadata:
  namespace: gmp-public  # TEM que ser aqui
  name: config
spec:
  managedAlertmanager:
    enabled: true
    configSecret:
      name: alertmanager-config
```

**Timeline de um Alerta**

Do pod cair até o email chegar:

```
00:00  Pod morre → up = 0
00:30  Collector faz scrape → detecta up=0
01:00  Rule Evaluator executa query → marca PENDING
02:00  Passou 1min (for: 1m) → muda pra FIRING
02:10  AlertManager aguarda group_wait (10s)
02:10  Email enviado via SMTP
02:15  Email chega na caixa de entrada
```

Total: ~2-3 minutos. É o esperado, não é bug!

**Key Takeaways**

✅ GMP abstrai complexidade de storage, scaling e HA do Prometheus
✅ Métricas de infra (CPU/mem) já existem via kubelet - não precisa instrumentar
✅ Métricas de app (request rate, latência) precisam de instrumentação
✅ Use `sum(up{...}) < 1` ao invés de `or absent()` em alertas
✅ CRDs são a interface: PodMonitoring (coleta), Rules (alertas), OperatorConfig (config global)
✅ AlertManager gerenciado incluso - só configurar SMTP no Secret

**Custos**

O GMP em si é grátis até 50GB/mês de métricas (você dificilmente chega nisso em labs).

Você só paga o cluster GKE: ~$50-75/mês no GKE Autopilot.

**Repositório**

Documentei tudo com exemplos práticos, troubleshooting e armadilhas comuns:
[link do seu repo]

Inclui:
- Arquitetura detalhada dos componentes
- Guia de métricas: kubelet vs aplicação
- Setup passo a passo (~15min)
- Debug e API do AlertManager

---

Qual foi sua maior dificuldade implementando observabilidade no K8s? 👇

#Kubernetes #Observability #Prometheus #GCP #SRE #DevOps #CloudNative

---

## Versão 2: Mais Direta (post médio) - COM ARQUITETURA E API

---

**Google Managed Prometheus: O que aprendi implementando alertas no GKE**

Implementei um sistema de observabilidade completo usando GMP e descobri 3 coisas que ninguém te conta:

**1️⃣ Você provavelmente não precisa instrumentar sua app**

80% dos alertas de infra usam métricas que já existem:
- CPU/Memória → kubelet (cAdvisor)
- Status de pods → kube-state-metrics
- Deployments/ReplicaSets → kube-state-metrics

Exemplo de alerta de memória alta SEM instrumentação:
```yaml
expr: |
  (container_memory_working_set_bytes{container!=""}
  / container_spec_memory_limit_bytes{container!=""}) > 0.85
```

Você só precisa instrumentar (expor `/metrics`) para:
- Request rate / latência
- Métricas de negócio (vendas, filas, etc)
- Golden Signals específicos da app

**2️⃣ NUNCA use `or absent()` em queries de alerta**

Query que parece correta mas quebra tudo:
```yaml
# ❌ Alerta dispara mas NUNCA resolve
expr: up{job="myapp"} == 0 or absent(up{job="myapp"})
```

Por quê? O `absent()` pode retornar true mesmo com pods ativos.

Solução que funciona:
```yaml
# ✅ Dispara E resolve
expr: sum(up{job="myapp"}) < 1
```

**3️⃣ A API do AlertManager é essencial pra debug**

Quando o alerta não chega, não fica adivinhando. O AlertManager tem uma API REST que mostra tudo:

```bash
# Port-forward pro AlertManager
kubectl port-forward -n gmp-system svc/alertmanager 9093:9093

# Ver alertas ativos (FIRING, PENDING, RESOLVED)
curl http://localhost:9093/api/v2/alerts

# Ver config aplicada (SMTP, receivers, routes)
curl http://localhost:9093/api/v2/status
```

Já me salvou várias vezes:
- Email não chegava? API mostrou que o alerta tava em "suppressed" (silenciado)
- Alerta não disparava? API mostrou que a query não matchava nada
- Config SMTP errada? API mostrou erro de autenticação

**Arquitetura: Como tudo se conecta**

O GMP não é uma caixa preta. Tem 4 componentes rodando no seu cluster:

```
┌─────────────────────────────────────────────┐
│              Seu Cluster GKE                │
│                                             │
│  ┌─────────┐  ┌─────────┐                  │
│  │  Pod 1  │  │  Pod 2  │                  │
│  │/metrics │  │/metrics │                  │
│  └────┬────┘  └────┬────┘                  │
│       │            │                        │
│  ┌────▼────────────▼─────┐  (gmp-system)   │
│  │  1. Collector          │                 │
│  │  Scrape a cada 30s     │                 │
│  └────────┬───────────────┘                 │
│           │                                 │
│  ┌────────▼───────────────┐                 │
│  │  2. Rule Evaluator     │                 │
│  │  Executa PromQL        │                 │
│  └────────┬───────────────┘                 │
│           │                                 │
│  ┌────────▼───────────────┐                 │
│  │  3. AlertManager       │                 │
│  │  Agrupa/Roteia         │                 │
│  └────────┬───────────────┘                 │
└───────────┼─────────────────────────────────┘
            │
            ▼
    ┌───────────────┐
    │  Gmail SMTP   │
    │  ou SendGrid  │
    └───────┬───────┘
            │
            ▼
          📧 Email
```

**O que cada componente faz:**

1. **Collector** (DaemonSet)
   - Lê os CRDs `PodMonitoring` que você cria
   - Faz scrape dos endpoints /metrics
   - Envia dados pro backend gerenciado do GMP

2. **Rule Evaluator** (Deployment)
   - Lê os CRDs `Rules` que você cria
   - Executa queries PromQL contra o backend
   - Marca alertas como PENDING → FIRING → RESOLVED

3. **AlertManager** (StatefulSet)
   - Recebe alertas do Rule Evaluator
   - Agrupa alertas similares (evita spam)
   - Envia pra receivers (Email, Slack, PagerDuty)
   - Expõe API REST na porta 9093 (debug!)

4. **Operator** (Deployment)
   - Gerencia o ciclo de vida dos CRDs
   - Reconcilia estado desejado vs atual

**Timeline: Do pod cair até o email chegar**

```
T+0s    Pod morre → up = 0
T+30s   Collector detecta → envia pro GMP
T+60s   Rule Evaluator: passou 1min (for: 1m) → FIRING
T+70s   AlertManager: aguarda group_wait (10s) → envia email
T+75s   📧 Email chega
```

Total: ~75 segundos. É o esperado, não é bug!

**3 CRDs que você precisa conhecer:**

`PodMonitoring` → O que coletar (selector, porta, intervalo)
`Rules` → Quando alertar (PromQL, for, labels)
`OperatorConfig` → Config global (SMTP, receivers) - TEM que estar em `gmp-public`!

**Por que GMP vs Prometheus self-hosted?**

✅ Storage gerenciado (24 meses retenção)
✅ Scaling automático
✅ Alta disponibilidade inclusa
✅ AlertManager gerenciado com API
✅ 50GB/mês grátis

Você só paga o cluster GKE (~$50-75/mês).

**Vamos ver isso funcionando na prática?**

Simulei um problema real: escalar o app pra 0 réplicas (como se tivesse crashado).

```bash
# T+0s: Derrubo todos os pods
kubectl scale deployment sample-app --replicas=0

# Pods começam a terminar...
```

**O que acontece nos bastidores:**

```
T+0s    kubectl scale --replicas=0
        └─ Pods começam a terminar

T+30s   Collector faz scrape
        └─ Detecta up=0
        └─ Envia pro backend GMP

T+60s   Rule Evaluator executa query
        └─ sum(up{...}) < 1 → true há 1min
        └─ Alerta muda: PENDING → FIRING

T+70s   AlertManager processa
        └─ Aguarda group_wait (10s)
        └─ Agrupa alertas similares
        └─ Envia via SMTP

T+75s   📧 Email chega!
        Subject: [FIRING:1] PodDown default/sample-app
```

**Como validei?** Usando a API do AlertManager:

```bash
kubectl port-forward -n gmp-system svc/alertmanager 9093:9093
curl http://localhost:9093/api/v2/alerts | jq
```

Retorna o alerta ativo:
```json
{
  "labels": {
    "alertname": "PodDown",
    "severity": "critical"
  },
  "status": {
    "state": "active"
  },
  "startsAt": "2025-10-25T20:05:00Z"
}
```

[Imagem 1: Pods do GMP rodando (gmp-system)]
[Imagem 2: App escalado pra 0 - pods em Terminating]
[Imagem 3: JSON do alerta na API do AlertManager]
[Imagem 4: Email chegando com [FIRING:1] PodDown]

Total do fluxo: **~75 segundos** do problema até a notificação. Rápido o suficiente pra reagir, mas não tão rápido que gera falso positivo.

**Repositório**

Documentei tudo com exemplos práticos, troubleshooting e API endpoints: [link do repo]

Inclui:
- Setup completo (~15min)
- Exemplos de queries que funcionam
- Como debugar com a API do AlertManager
- Comandos pra reproduzir essa demo
- Armadilhas comuns e soluções

Qual sua experiência com Prometheus gerenciado? Já usou a API do AlertManager pra debug? 👇

#Kubernetes #GCP #Observability #SRE #DevOps

---

## Versão 3: Thread (vários posts)

### Post 1: O Problema

---

**Por que gerenciar Prometheus self-hosted é uma dor de cabeça?**

Se você já rodou Prometheus em produção, sabe os problemas:

❌ Storage persistente (PV, escalabilidade, custos)
❌ Alta disponibilidade (múltiplas réplicas, thanos)
❌ Backup e disaster recovery
❌ Se o Prometheus cai, você fica cego
❌ Scaling manual quando o volume cresce

E o pior: você passa mais tempo gerenciando o sistema de observabilidade do que usando ele.

Testei o Google Managed Prometheus e mudou o jogo. 🧵

#Kubernetes #Observability #GCP

---

### Post 2: A Solução

---

**Google Managed Prometheus: Prometheus gerenciado que roda no seu cluster**

Como funciona:

O GMP instala 4 componentes no namespace `gmp-system`:

🔹 **Collector** (DaemonSet)
Faz scrape dos pods e envia pro backend gerenciado

🔹 **Rule Evaluator**
Executa queries PromQL e dispara alertas

🔹 **AlertManager**
Agrupa e roteia notificações (Email/Slack/PagerDuty)

🔹 **Operator**
Gerencia os CRDs e reconcilia configurações

O backend (storage, query engine, HA) é completamente gerenciado pelo Google.

Você só cria os YAMLs e pronto.

Custo: Grátis até 50GB/mês. Você só paga o cluster GKE.

Próximo post: A descoberta que mudou como penso sobre métricas 👇

#Kubernetes #SRE #DevOps

---

### Post 3: A Descoberta

---

**Você NÃO precisa instrumentar sua app para alertas de infraestrutura**

Essa foi a descoberta mais importante.

Existem 3 tipos de métricas no K8s:

**1. Métricas do kubelet (automáticas)**
CPU, memória, disco, rede
Coletadas via cgroups
✅ Já existem no GMP

**2. Métricas do kube-state-metrics (automáticas)**
Status de pods, deployments, nodes
Lê da API do K8s
✅ Já existem no GMP

**3. Métricas da aplicação (manual)**
Request rate, latência, métricas de negócio
❌ Precisa instrumentar (Prometheus client)

Exemplo prático:

Alerta de memória alta → Usa kubelet (automático)
Alerta de taxa de erro → Usa app (instrumentação)

Próximo post: O erro que me custou horas de debug 👇

#Kubernetes #Observability #Prometheus

---

### Post 4: O Erro Crítico

---

**NUNCA use `or absent()` em queries de alerta no Prometheus**

Gastei horas debugando isso.

A query que parecia correta:

```yaml
expr: up{job="myapp"} == 0 or absent(up{job="myapp"})
```

O que aconteceu:
✅ Pods caem → alerta DISPARA
❌ Pods voltam → alerta CONTINUA ATIVO

Por quê?
O `absent()` pode retornar true mesmo com pods ativos se houver séries temporais antigas no TSDB.

**Solução:**

```yaml
expr: sum(up{job="myapp"}) < 1
```

Simples, confiável, resolve automaticamente.

Key lesson: Queries PromQL precisam ser **reversíveis**. Se a condição para disparar é X, a condição para resolver precisa ser ¬X.

Documentei isso e mais armadilhas no repo: [link]

Qual foi o bug mais difícil que você já encontrou em queries PromQL? 👇

#Prometheus #Observability #SRE #PromQL

---

## Dicas de Publicação

**Melhor formato:**
- LinkedIn privilegia posts nativos (não links externos no início)
- Coloque o link do repo no primeiro comentário ou no final
- Use quebras de linha para facilitar leitura
- Emojis com moderação (apenas para destacar seções)

**Horário:**
- Terça a quinta, 8h-10h ou 17h-19h (horário BR)
- Evite segunda de manhã e sexta à tarde

**Engagement:**
- Responda todos os comentários nas primeiras 2 horas
- Faça perguntas no final (call to action)
- Compartilhe em grupos relevantes (Kubernetes BR, SRE Brasil, etc)

**Hashtags:**
Use 3-5 hashtags relevantes:
- #Kubernetes (2.5M seguidores)
- #DevOps (3M seguidores)
- #SRE (relevante mas menor)
- #CloudNative (relevante)
- #GCP ou #GoogleCloud (se focado em GCP)

**Formato recomendado:**
Versão 2 (post médio) tem melhor custo-benefício:
- Técnico o suficiente pra agregar valor
- Curto o suficiente pra segurar atenção
- Com código pra demonstrar expertise
- Com call to action no final

Quer que eu ajuste algo ou crie uma versão customizada?
