# Guia de Aprendizado: Observabilidade com Prometheus

Esse documento explica os conceitos por trás do projeto. Se você só quer fazer funcionar, vai direto pro `gcp/setup-guide.md` ou `aws/setup-guide.md`.

## Por que observabilidade?

Quando você sobe uma aplicação em produção, precisa saber:
- Ela tá funcionando?
- Tem algum pod travado?
- Memória/CPU tão estourando?
- Tem erro acontecendo?

Sem observabilidade, você só descobre quando usuário reclama. Com observabilidade, você sabe antes.

## O que é Prometheus?

Prometheus é um sistema de monitoramento. Ele:

1. **Coleta métricas** dos seus pods a cada X segundos (scraping)
2. **Armazena** essas métricas num banco de séries temporais (TSDB)
3. **Avalia regras** pra ver se algo tá errado
4. **Dispara alertas** quando identifica problema

### Como funciona a coleta?

Seu app expõe um endpoint `/metrics` com dados tipo:

```
# Quantas requisições teve
http_requests_total{method="GET",path="/api/users"} 1523

# Quantos pods tão ativos
up{job="sample-app",pod="sample-app-abc123"} 1

# Uso de memória
container_memory_working_set_bytes{container="app"} 134217728
```

O Prometheus varre esses endpoints periodicamente e salva tudo.

### PromQL: a linguagem de query

Pra buscar métricas, você usa PromQL:

```promql
# Quantos pods tão up?
up{job="sample-app"}

# Taxa de requisições por segundo
rate(http_requests_total[5m])

# Uso de memória em %
(container_memory_working_set_bytes / container_spec_memory_limit_bytes) * 100
```

## Alerting: como saber quando tem problema

Você define regras de alerta:

```yaml
- alert: PodDown
  expr: up{job="sample-app"} == 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Pod não tá respondendo"
    description: "Pod {{ $labels.pod }} tá fora há mais de 1 minuto"
```

O que isso faz:
1. A cada X segundos, o Prometheus avalia `up{job="sample-app"} == 0`
2. Se der true, o alerta entra em **PENDING**
3. Se continuar true por 1 minuto (`for: 1m`), vira **FIRING**
4. Quando vira FIRING, o AlertManager é notificado

O `for` existe pra evitar alerta por ruído momentâneo. Se o pod reiniciar rápido (health check falhou mas já voltou), não dispara.

### AlertManager: o roteador de alertas

O AlertManager recebe os alertas do Prometheus e decide o que fazer:

- **Agrupamento**: Se 10 pods caírem juntos, agrupa num email só
- **Silenciamento**: Ignora alertas durante manutenção programada
- **Roteamento**: Alerta critical vai pro PagerDuty, warning vai pro email
- **Retry**: Se o email falhar, tenta de novo

No nosso setup, o AlertManager pega o alerta e manda pro SMTP (Gmail).

## Managed vs Self-Hosted

### Self-Hosted (tradicional)

Você instala o Prometheus no cluster:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        volumeMounts:
        - name: storage
          mountPath: /prometheus
```

Problemas:
- Precisa gerenciar storage (se o disco encher, perde métricas)
- Escalabilidade limitada (1 pod só não aguenta muito)
- Alta disponibilidade complexa (precisa de replica, shared storage, etc)
- Backup é problema seu

### Managed (AWS AMP / Google GMP)

O cloud provider cuida de tudo. Você só configura o que coletar e as regras.

No GCP:
- Você cria um `PodMonitoring` (CRD) dizendo quais pods coletar
- O GMP automaticamente coleta, armazena, avalia regras
- Você não vê pod do Prometheus rodando no cluster

No AWS:
- Você instala um agent que faz remote write pro AMP
- O AMP roda fora do cluster
- Você também não gerencia nada

Vantagens:
- Escala automático (milhões de séries temporais)
- Storage ilimitado (você paga por uso)
- Alta disponibilidade inclusa
- Backup automático

Desvantagens:
- Custo (mas geralmente compensa vs gerenciar você mesmo)
- Vendor lock-in (migrar é chato)

## Anatomia de uma métrica Prometheus

```
metric_name{label1="value1",label2="value2"} 42
```

**metric_name**: O que você tá medindo (ex: `http_requests_total`)

**labels**: Dimensões da métrica (ex: `method="GET"`, `status="200"`)

**valor**: O número em si

### Tipos de métricas

**Counter**: Só aumenta (ou reseta pra 0). Ex: total de requisições.
```promql
# Taxa de crescimento nos últimos 5 min
rate(http_requests_total[5m])
```

**Gauge**: Pode subir e descer. Ex: uso de memória, temperatura.
```promql
# Valor atual
container_memory_usage_bytes
```

**Histogram**: Distribui valores em buckets. Ex: latência de requisições.
```promql
# 95% das requisições levam até quanto tempo?
histogram_quantile(0.95, http_request_duration_seconds_bucket)
```

**Summary**: Parecido com histogram, mas calcula quantis no cliente.

## Google Managed Prometheus (GMP): como funciona?

Quando você habilita GMP no GKE (`--enable-managed-prometheus`), o Google cria:

### Namespace gmp-system

Pods que o Google gerencia:
- **collector**: Scrapa as métricas dos seus pods
- **rule-evaluator**: Avalia as regras de alerta
- **frontend**: API query compatível com Prometheus

Você não edita esses pods. Eles são gerenciados automaticamente.

### Namespace gmp-public

Onde você coloca suas configs:
- **OperatorConfig**: Config global (AlertManager, retention, etc)
- **Rules**: Regras de alerta
- **PodMonitoring**: O que coletar e onde

### Como o collector acha seus pods?

Você cria um `PodMonitoring`:

```yaml
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: meu-app
  namespace: default
spec:
  selector:
    matchLabels:
      app: meu-app
  endpoints:
  - port: metrics
    interval: 30s
```

O collector automaticamente:
1. Lista pods com label `app: meu-app`
2. A cada 30s, faz GET em `http://<pod-ip>:<metrics-port>/metrics`
3. Parseia as métricas
4. Envia pro backend do GMP (fora do cluster)

### Onde as métricas ficam?

No Cloud Monitoring do GCP. Você pode:
- Ver no Metrics Explorer (console web)
- Fazer query via API
- Criar dashboards
- (Não dá pra) acessar direto via `kubectl port-forward` como no Prometheus tradicional

### Como funciona o AlertManager gerenciado?

Você cria um Secret com a config:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-config
  namespace: gmp-public
stringData:
  alertmanager.yaml: |
    global:
      smtp_from: 'seu@email.com'
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_auth_username: 'seu@email.com'
      smtp_auth_password: 'sua-app-password'
    route:
      receiver: 'email-alerts'
    receivers:
    - name: 'email-alerts'
      email_configs:
      - to: 'destino@email.com'
```

E aponta o OperatorConfig pra ele:

```yaml
apiVersion: monitoring.googleapis.com/v1
kind: OperatorConfig
metadata:
  name: config
  namespace: gmp-public
spec:
  managedAlertmanager:
    configSecret:
      name: alertmanager-config
      key: alertmanager.yaml
```

O GMP automaticamente:
1. Lê o Secret
2. Configura o AlertManager gerenciado (que roda fora do cluster)
3. Quando um alerta dispara, o AlertManager manda email via SMTP

Você não vê pod do AlertManager no cluster. Tudo roda como serviço gerenciado.

## Fluxo completo de um alerta

1. **App expõe métrica**: `/metrics` retorna `up{job="sample-app"} 1`
2. **Collector scrapa**: A cada 30s, pega a métrica
3. **Métrica é armazenada**: No Cloud Monitoring
4. **Rule evaluator avalia**: A cada 30s (intervalo da rule), roda `up{job="sample-app"} == 0`
5. **Alerta vira PENDING**: Query deu true
6. **Alerta vira FIRING**: Passou o tempo do `for: 1m` e ainda tá true
7. **AlertManager recebe**: Rule evaluator notifica o AlertManager
8. **AlertManager agrupa**: Espera `group_wait: 10s` pra agrupar alertas similares
9. **AlertManager envia**: Faz SMTP pro Gmail
10. **Gmail envia**: Email chega na caixa de entrada

Tempo total: ~2-3 minutos desde o pod cair até o email chegar.

## Namespace: por que importa?

No GMP, o namespace é importante:

**gmp-public**: Onde o GMP procura configs (Rules, OperatorConfig, Secrets)

**gmp-system**: Onde ficam os pods gerenciados (você não meche)

**default** (ou qualquer outro): Onde ficam seus apps

Se você colocar o Rules no namespace errado:

```yaml
# ERRADO
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: meu-alert
  namespace: default  # ❌
```

O GMP não vai achar. Tem que ser:

```yaml
# CERTO
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: meu-alert
  namespace: gmp-public  # ✅
```

Mas o `PodMonitoring` pode ficar em qualquer namespace (geralmente fica junto com o app).

## SMTP: como funciona o envio de email?

SMTP (Simple Mail Transfer Protocol) é o protocolo pra enviar email. Funciona assim:

1. **Conecta** no servidor SMTP (ex: `smtp.gmail.com:587`)
2. **Autentica** com usuário/senha
3. **Envia** o email (from, to, subject, body)
4. **Desconecta**

O AlertManager faz isso quando precisa enviar um alerta.

### Por que App Password?

Google (e outros) não deixam apps se conectarem com sua senha normal. Você precisa gerar uma App Password, que é uma senha específica só pra esse app.

Se você desabilitar a App Password depois, o AlertManager vai começar a falhar ao enviar emails.

### Alternativas ao SMTP

Além de email via SMTP, o AlertManager suporta:

- **Webhook**: Manda POST pra uma URL (você processa como quiser)
- **Slack**: Notificação direto no Slack
- **PagerDuty**: Integração com sistema de on-call
- **Opsgenie**: Outro sistema de on-call
- **VictorOps**: Mais um sistema de on-call
- **Discord**: Via webhook

No nosso setup, usamos SMTP porque é mais simples e não depende de outros serviços.

## Troubleshooting: guia mental

**Métricas não aparecem?**
- Problema na coleta (PodMonitoring errado, pod sem label, porta errada)

**Alerta não dispara?**
- Problema na regra (PromQL errada, `for` muito alto, namespace errado)

**Email não chega?**
- Problema no AlertManager (SMTP errado, senha errada, firewall)

Pra debugar, sempre vá do início pro fim:
1. Métrica tá sendo exposta? → `curl http://pod-ip:port/metrics`
2. Collector tá coletando? → Logs do collector, ou Metrics Explorer
3. Regra tá correta? → `kubectl get rules -o yaml`, query no Metrics Explorer
4. AlertManager tá configurado? → `kubectl get secret`, verifica SMTP

## Custos: como calcular?

**GKE Autopilot**: Você paga por CPU/memória que os pods usam
- 2 pods com 100m CPU e 128Mi memória cada: ~$50-70/mês

**GKE Standard**: Você paga pelos nodes + control plane
- 2x e2-medium (2 vCPU, 4GB cada): ~$50/mês
- Control plane: $73/mês
- Total: ~$123/mês

**GMP**: Grátis até 50GB métricas/mês
- Esse lab usa ~100MB/mês
- Custo: $0

**SMTP**: Gmail é grátis até 500 emails/dia
- Você vai mandar ~10-20 emails/mês testando
- Custo: $0

**Total**: Basicamente o custo do cluster. A observabilidade em si é grátis.

## Próximos passos

Se você entendeu os conceitos, pode:

1. **Adicionar alertas customizados**: Monitora métricas específicas da sua app
2. **Integrar com Slack**: Mais conveniente que email
3. **Criar dashboards**: Visualizar métricas em tempo real
4. **Configurar severity routing**: Critical vai pro PagerDuty, warning pro Slack
5. **Monitorar infra**: Node down, disk cheio, etc

Recursos úteis:
- Documentação PromQL: https://prometheus.io/docs/prometheus/latest/querying/basics/
- Awesome Prometheus Alerts: https://samber.github.io/awesome-prometheus-alerts/
- GMP docs: https://cloud.google.com/stackdriver/docs/managed-prometheus
