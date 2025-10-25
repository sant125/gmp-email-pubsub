# Debug e API do GMP

## Endpoints úteis

O AlertManager roda em `gmp-system` e expõe uma API REST na porta 9093.

### Acessar o AlertManager

```bash
# Port-forward pro AlertManager
kubectl port-forward -n gmp-system svc/alertmanager 9093:9093

# Em outra aba, usa a API
curl http://localhost:9093/api/v2/alerts
```

### Principais endpoints

**Ver alertas ativos:**
```bash
curl http://localhost:9093/api/v2/alerts
```

Retorna JSON com todos os alertas, incluindo:
- `status.state`: "active", "suppressed" ou "unprocessed"
- `startsAt`: quando o alerta começou
- `annotations`: descrição do alerta
- `labels`: severity, namespace, pod, etc

**Ver status do AlertManager:**
```bash
curl http://localhost:9093/api/v2/status
```

Mostra:
- Config carregada (com senhas mascaradas como `<secret>`)
- Versão do AlertManager
- Status do cluster
- Uptime

**Ver silences ativos:**
```bash
curl http://localhost:9093/api/v2/silences
```

**Criar um silence:**
```bash
curl -X POST http://localhost:9093/api/v2/silences -d '{
  "matchers": [
    {
      "name": "alertname",
      "value": "PodDown",
      "isRegex": false
    }
  ],
  "startsAt": "2025-10-25T20:00:00Z",
  "endsAt": "2025-10-25T22:00:00Z",
  "createdBy": "seu-nome",
  "comment": "Deploy em andamento"
}'
```

**Testar a config do AlertManager:**
```bash
# Pega a config atual
kubectl get secret alertmanager-config -n gmp-public -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d

# Não tem como validar antes de aplicar, então testa direto
kubectl apply -f alertmanager-config.yaml

# Checa se foi aplicada
kubectl logs -n gmp-system alertmanager-0 -c alertmanager --tail=10
```

### Verificar regras de alerta

```bash
# Lista todas as Rules
kubectl get rules -A

# Detalha uma rule específica
kubectl get rules lab-alerts -n default -o yaml

# O campo importante é status.conditions
# ConfigurationCreateSuccess: True = regra foi aceita
# Se der False, tem erro de sintaxe
```

### Consultar métricas manualmente

O GMP não expõe uma API de query direta como o Prometheus normal. Precisa usar o Cloud Monitoring.

**Via gcloud:**
```bash
gcloud monitoring time-series list \
  --filter='metric.type="prometheus.googleapis.com/up/gauge"' \
  --format=json
```

**Via Console:**
https://console.cloud.google.com/monitoring/metrics-explorer

Aí você monta a query PromQL:
```promql
up{job="default/sample-app-monitoring"}
```

## Problemas que encontramos

### 1. ⚠️ NUNCA use `absent()` em alertas que precisam resolver

**Problema CRÍTICO:**
O `or absent(up{...})` impede que o alerta **resolva** quando os pods voltam!

Query problemática:
```yaml
# ❌ ERRADO - Alerta nunca resolve
expr: up{job="default/sample-app-monitoring"} == 0 or absent(up{job="default/sample-app-monitoring"})
```

**Por que não funciona:**
- Quando pods estão down: `up == 0` → alerta DISPARA ✅
- Quando pods voltam: `up == 1`, mas `absent(up)` pode ainda retornar `true` se houver séries temporais antigas
- Resultado: alerta fica **ativo pra sempre**

**Solução CORRETA:**
Use `sum()` para agregar todos os pods:

```yaml
# ✅ CORRETO - Alerta dispara E resolve
expr: sum(up{job="default/sample-app-monitoring"}) < 1
```

**Como funciona:**
- Quando todos os pods estão down: `sum(up) = 0` → dispara ✅
- Quando pelo menos 1 pod volta: `sum(up) >= 1` → resolve ✅
- Simples, confiável e eficiente

**Alternativa (usando kube-state-metrics):**
```yaml
# Alerta quando pod não está Running
expr: kube_pod_status_phase{namespace="default", pod=~"sample-app-.*", phase!="Running"} == 1
```

**Quando usar `absent()`:**
Apenas em queries de observação, nunca em alertas que precisam resolver automaticamente.

### 2. Namespace das Rules importa

**Problema:**
Criamos as Rules em vários namespaces diferentes (gmp-public, default, etc) e algumas não funcionavam.

**Solução:**
Pode criar Rules em qualquer namespace, mas o job name precisa incluir o namespace do target:

```yaml
# Rule em default, monitorando pod em default
expr: up{job="default/sample-app-monitoring"}

# Se criar Rule em gmp-public, precisa especificar namespace
expr: up{namespace="default",job="sample-app-monitoring"}
```

A gente deixou em `default` mesmo porque é onde o app roda.

### 3. AlertManager config tem que estar em gmp-public

**Problema:**
Tentamos criar o Secret do AlertManager no namespace `default` e não funcionou.

**Solução:**
O GMP só olha configs no namespace `gmp-public`:

```bash
kubectl create secret generic alertmanager-config \
  --from-file=alertmanager.yaml \
  -n gmp-public
```

E o OperatorConfig também precisa estar lá:

```yaml
apiVersion: monitoring.googleapis.com/v1
kind: OperatorConfig
metadata:
  namespace: gmp-public  # TEM que ser aqui
  name: config
```

### 4. Gmail App Password com espaços

**Problema:**
O Google gera a App Password com espaços tipo: `abcd efgh ijkl mnop`. A gente ficou em dúvida se podia colocar com espaços no YAML.

**Solução:**
Tanto faz, funciona com ou sem espaços:

```yaml
# Funciona
smtp_auth_password: 'abcd efgh ijkl mnop'

# Também funciona
smtp_auth_password: 'abcdefghijklmnop'
```

### 5. Labels no matcher do alert

**Problema:**
A query `absent(up{job="sample-app"})` não matchava nada, alerta nunca disparava.

**Solução:**
O job name no GMP sempre tem o formato `namespace/podmonitoring-name`:

```yaml
# PodMonitoring
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: sample-app-monitoring
  namespace: default

# A métrica fica assim
up{job="default/sample-app-monitoring"}

# Então o alert precisa usar o nome completo
expr: up{job="default/sample-app-monitoring"} == 0
```

### 6. Logs do AlertManager não aparecem

**Problema:**
Quando o email não chegava, a gente queria ver os logs do AlertManager pra debugar. Mas o pod tem 2 containers.

**Solução:**
Precisa especificar qual container:

```bash
# Errado (mostra o config-reloader)
kubectl logs -n gmp-system alertmanager-0

# Certo
kubectl logs -n gmp-system alertmanager-0 -c alertmanager
```

### 7. Alerts chegam com delay

**Problema:**
Derrubamos os pods e esperamos o email chegar na hora.

**Solução:**
Tem delay natural no fluxo:

1. Pod morre → métrica `up` vira 0 imediatamente
2. Collector do GMP coleta métricas a cada 30s (default)
3. Rule evaluator roda a cada 30s (configurável em `interval`)
4. Alerta fica pending por 1min (nosso `for: 1m`)
5. Alerta vira FIRING → vai pro AlertManager
6. AlertManager agrupa e espera 10s (`group_wait: 10s`)
7. Email é enviado

No total: **~2-3 minutos** desde o pod morrer até o email chegar. É normal.

### 8. Email cai no spam

**Problema:**
Os primeiros emails do AlertManager caíram direto no spam.

**Solução:**
Marca como "Não é spam" uma vez que os próximos já vêm na caixa de entrada. O Gmail aprende.

Ou configura um filtro:
```
De: seu-email@gmail.com
Assunto: contém "[FIRING]"
Fazer: Nunca enviar para spam
```

## Testando manualmente

### Forçar um alerta

```bash
# Derruba os pods
kubectl scale deployment sample-app --replicas=0

# Espera 2-3 minutos
# Email deve chegar

# Volta os pods
kubectl scale deployment sample-app --replicas=2

# Se send_resolved: true, recebe outro email
```

### Simular alta memória

```bash
# Exec no pod
kubectl exec -it deployment/sample-app -- sh

# Aloca memória (não funciona nessa imagem)
# dd if=/dev/zero of=/dev/null bs=1M count=100

# Essa imagem não tem ferramentas pra simular, então vai precisar
# deployar um pod diferente tipo stress-ng
```

### Ver se métrica tá sendo coletada

Vai no Metrics Explorer:
https://console.cloud.google.com/monitoring/metrics-explorer

Query:
```promql
up{job="default/sample-app-monitoring"}
```

Se aparecer gráfico = tá coletando.
Se não aparecer = problema no PodMonitoring.

## Ferramentas úteis

**amtool** (AlertManager CLI):

Não vem instalado, mas dá pra baixar:
```bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
tar xvf alertmanager-0.27.0.linux-amd64.tar.gz
./amtool --alertmanager.url=http://localhost:9093 alert
```

**promtool** (validar regras):

```bash
# Extrai as regras
kubectl get rules lab-alerts -o jsonpath='{.spec}' > rules.yaml

# Valida (precisa ter promtool instalado)
promtool check rules rules.yaml
```

Mas não funciona 100% porque o GMP usa extensões custom.
