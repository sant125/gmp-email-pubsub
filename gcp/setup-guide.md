# Setup GCP - Google Managed Prometheus

## Como funciona

```
Pod → GMP Collector → Managed Prometheus → AlertManager → SMTP → Email
```

É só YAML e kubectl. Sem serverless, sem pub/sub, sem código custom.

## Google Managed Prometheus

É Prometheus gerenciado. Você não gerencia pods, storage, nada. O Google faz isso.

Bom:
- Sem manutenção
- Escala automaticamente
- 50GB/mês grátis (você não vai chegar nisso)
- AlertManager incluso

Ruim:
- Precisa SMTP externo (usei Gmail)
- Meio preso ao GKE

## Pré-requisitos

- Conta GCP ativa
- gcloud CLI instalado
- kubectl instalado
- Gmail ou outro SMTP

## 1. Setup inicial

```bash
export PROJECT_ID="seu-project"
export CLUSTER_NAME="lab-cluster"

gcloud config set project $PROJECT_ID

# Habilita APIs necessárias
gcloud services enable container.googleapis.com \
  monitoring.googleapis.com
```

## 2. Cria cluster GKE com GMP

**Opção A: GKE Autopilot (recomendado - mais barato)**
```bash
gcloud container clusters create-auto $CLUSTER_NAME \
  --region=us-central1 \
  --enable-managed-prometheus
```

**Opção B: GKE Standard**
```bash
gcloud container clusters create $CLUSTER_NAME \
  --zone us-central1-a \
  --num-nodes 2 \
  --machine-type e2-medium \
  --enable-managed-prometheus \
  --workload-pool=$PROJECT_ID.svc.id.goog
```

O `--enable-managed-prometheus` é o que importa. Ele cria automaticamente o namespace `gmp-system` com os collectors e o `gmp-public` onde você coloca suas configs.

```bash
# Pega credenciais
gcloud container clusters get-credentials $CLUSTER_NAME --region=us-central1  # Autopilot
# OU
gcloud container clusters get-credentials $CLUSTER_NAME --zone=us-central1-a  # Standard

# Verifica se GMP tá ativo
kubectl get ns gmp-public gmp-system
kubectl get pods -n gmp-system
```

Você vai ver pods do tipo `collector`, `rule-evaluator` e outros. Esses são os componentes do GMP que o Google gerencia.

## 3. Gmail App Password

AlertManager precisa de SMTP. Gmail funciona.

1. https://myaccount.google.com/apppasswords
2. Cria App Password nova
3. Copia os 16 caracteres (ex: `abcd efgh ijkl mnop`)
4. Guarda

Não pode usar sua senha normal, Google bloqueia. App Password é específica pra isso.

## 4. Configura AlertManager

```bash
cd gcp/k8s-manifests

# Edita alertmanager-config.yaml e substitui:
# - smtp_from: coloca teu email
# - smtp_auth_username: coloca teu email de novo
# - smtp_auth_password: cola a App Password (com ou sem espaços, tanto faz)
# - to: email que vai receber os alertas
```

O arquivo tem dois recursos:
1. **Secret**: onde ficam as configs do AlertManager (SMTP, templates de email, etc)
2. **OperatorConfig**: diz pro GMP usar esse Secret como config do AlertManager

```bash
kubectl apply -f alertmanager-config.yaml

# Verifica
kubectl get secret alertmanager-config -n gmp-public
kubectl get operatorconfig config -n gmp-public -o yaml | grep -A 3 "managedAlertmanager"
```

Importante: o namespace tem que ser `gmp-public`. É onde o GMP espera encontrar as configs.

## 5. Deploy app de exemplo

```bash
kubectl apply -f sample-app.yaml

# Aguarda pods subirem
kubectl get pods -w
# Ctrl+C quando tiver 2 pods Running
```

O sample-app é só um app de exemplo que expõe métricas em `/metrics`. Qualquer app que exponha métricas Prometheus funciona.

## 6. Configura coleta de métricas

```bash
kubectl apply -f pod-monitoring.yaml

# Verifica
kubectl get podmonitoring
```

O `PodMonitoring` é um CRD do GMP. Ele diz pro collector:
- Quais pods coletar (via label selector)
- Qual porta/path das métricas
- Intervalo de coleta

Depois de uns 30 segundos, as métricas já devem estar aparecendo no GMP.

## 7. Cria regras de alerta

```bash
kubectl apply -f alerting-rules.yaml

# Verifica
kubectl get rules -n gmp-public
kubectl describe rules lab-alerts -n gmp-public
```

As Rules também são um CRD do GMP. Aqui você define:
- **PodDown**: Se `up{job="sample-app"} == 0` por mais de 1min, dispara alerta critical
- **HighMemoryUsage**: Se uso de memória > 80% por mais de 2min, dispara alerta warning

O campo `status.conditions` deve mostrar `ConfigurationCreateSuccess: True`. Se não, tem algo errado.

## 8. Testa

Derruba os pods:

```bash
kubectl scale deployment sample-app --replicas=0
kubectl get pods -l app=sample-app  # verifica que sumiram
```

Timeline:
1. **T+0s**: Pod morre, `up` vira 0
2. **T+60s**: Alerta vira FIRING (por causa do `for: 1m`)
3. **T+70s**: AlertManager pega
4. **T+80s**: SMTP envia

Email deve chegar em 2-3min. Checa spam se não aparecer.

```bash
# Quando testar, volta os pods
kubectl scale deployment sample-app -n default --replicas=2
```

Se tiver o `send_resolved: true` (já está na config), você vai receber outro email quando os pods voltarem.

## 9. Endpoints da API

O AlertManager expõe API REST. Útil pra debug:

```bash
# Port-forward (se não tiver)
kubectl port-forward -n gmp-system svc/alertmanager 9093:9093

# Ver alertas ativos
curl http://localhost:9093/api/v2/alerts

# Ver status
curl http://localhost:9093/api/v2/status
```

Tem mais endpoints e exemplos no `DEBUG.md`.

## 10. Debug

### Alertmanager não envia email

```bash
# Verifica a config
kubectl get secret alertmanager-config -n gmp-public -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d

# Infelizmente não dá pra ver logs do AlertManager gerenciado
# O GMP não expõe esses logs diretamente
```

Erros comuns:
- **Senha errada**: Confere se copiou a App Password certinha
- **Email errado**: Tem que ser o mesmo email que gerou a App Password
- **Firewall**: Se o cluster tiver firewall, precisa liberar saída na porta 587

### Alertas não disparam

```bash
# Verifica se as Rules foram criadas
kubectl get rules -n gmp-public -o yaml

# Testa a query manualmente no Metrics Explorer do GCP
# https://console.cloud.google.com/monitoring/metrics-explorer
# Query: up{job="sample-app"}
```

Se a métrica não aparecer lá, o problema é na coleta, não no alerta.

### Métrica não aparece

```bash
# Verifica PodMonitoring
kubectl get podmonitoring -A -o yaml

# Checa se os pods têm as labels corretas
kubectl get pods --show-labels | grep sample-app

# Verifica logs do collector
kubectl logs -n gmp-system -l app.kubernetes.io/name=collector --tail=50
```

## Alternativas ao Gmail

Gmail tem limite de 500 emails/dia. Se precisar de mais:

**SendGrid** (100 emails/dia grátis):
```yaml
smtp_smarthost: 'smtp.sendgrid.net:587'
smtp_auth_username: 'apikey'
smtp_auth_password: 'SG.xxxxxx'
```

**Mailgun** (100 emails/dia grátis):
```yaml
smtp_smarthost: 'smtp.mailgun.org:587'
smtp_auth_username: 'postmaster@seu.dominio'
smtp_auth_password: 'sua-api-key'
```

**AWS SES** ($0.10 por 1000 emails):
```yaml
smtp_smarthost: 'email-smtp.us-east-1.amazonaws.com:587'
smtp_auth_username: 'AKIAXXXXX'
smtp_auth_password: 'sua-smtp-password'
```

## Custos aproximados

- **GKE Autopilot**: ~$50-70/mês (baseado em uso real dos pods)
- **GKE Standard**: ~$75/mês (2x e2-medium + control plane $73/mês)
- **GMP**: Grátis (até 50GB métricas/mês, você não vai chegar nisso num lab)
- **Gmail SMTP**: Grátis (até 500 emails/dia)

**Total**: ~$50-75/mês

A parte boa é que a observabilidade em si é grátis. Você só paga pelo cluster.

## Cleanup

```bash
# Deleta resources do cluster primeiro
kubectl delete -f k8s-manifests/

# Depois deleta o cluster
gcloud container clusters delete $CLUSTER_NAME \
  --region=us-central1 \
  --quiet

# Se for Standard, usa --zone em vez de --region
```

## Próximos passos

Coisas que dá pra fazer:
- Adicionar mais alertas (CPU alta, disco cheio, erros de aplicação)
- Configurar rotas diferentes por severity (critical vai pro PagerDuty, warning pro email)
- Integrar com Slack/Discord em vez de email
- Adicionar dashboards customizados no Cloud Monitoring

Exemplos de alertas prontos:
https://samber.github.io/awesome-prometheus-alerts/
