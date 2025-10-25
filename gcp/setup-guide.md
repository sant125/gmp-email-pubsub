# GCP Setup - Alertas via GKE (SMTP direto)

## Arquitetura

```
Prometheus (GMP) → Alertmanager (GMP) → Gmail SMTP → Email
```

**Setup zero código**: Apenas YAMLs + kubectl apply!

## Pré-requisitos

- Conta GCP ativa
- gcloud CLI instalado
- kubectl instalado
- Gmail (ou outro email SMTP) - GRÁTIS

## 1. Setup inicial

```bash
export PROJECT_ID="seu-project"
export CLUSTER_NAME="lab-cluster"

gcloud config set project $PROJECT_ID

# Habilita APIs necessárias
gcloud services enable container.googleapis.com \
  monitoring.googleapis.com
```

## 2. Cria cluster GKE

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

```bash
# Pega credenciais
gcloud container clusters get-credentials $CLUSTER_NAME --region=us-central1  # Autopilot
# OU
gcloud container clusters get-credentials $CLUSTER_NAME --zone=us-central1-a  # Standard

# Verifica GMP
kubectl get ns gmp-public gmp-system
```

## 3. Configura Gmail App Password

1. Acessa: https://myaccount.google.com/apppasswords
2. Cria nova App Password
3. Copia a senha de 16 caracteres

## 4. Configura Alertmanager

```bash
cd gcp

# Edita k8s-manifests/alertmanager-config.yaml:
# - Substitui 'seu-email@gmail.com' pelo teu email (3 lugares)
# - Substitui 'sua-app-password-aqui' pela App Password do Gmail

# Aplica
kubectl apply -f k8s-manifests/alertmanager-config.yaml

# Verifica
kubectl get operatorconfig config -n gmp-public
kubectl get secret alertmanager-config -n gmp-public
```

## 5. Deploy app de exemplo

```bash
kubectl apply -f k8s-manifests/sample-app.yaml

# Aguarda pods
kubectl wait --for=condition=ready pod -l app=sample-app --timeout=60s
kubectl get pods
```

## 6. Configura coleta de métricas

```bash
kubectl apply -f k8s-manifests/pod-monitoring.yaml

# Verifica
kubectl get podmonitoring
```

## 7. Cria regras de alerta

```bash
kubectl apply -f k8s-manifests/alerting-rules.yaml

# Verifica
kubectl get rules -A
```

## 8. Aguarda GMP processar (1-2 min)

```bash
# Checa se métricas estão sendo coletadas
kubectl logs -n gmp-system -l app.kubernetes.io/name=collector --tail=20

# Verifica se Alertmanager subiu
kubectl get pods -n gmp-system -l app.kubernetes.io/name=alertmanager
```

## 9. Testa o alerta

```bash
# Derruba o app pra disparar alerta
kubectl scale deployment sample-app --replicas=0

# Aguarda ~2-3 minutos

# Checa logs do Alertmanager
kubectl logs -n gmp-system -l app.kubernetes.io/name=alertmanager --tail=50 | grep -i "email"

# Email deve chegar em 2-4 minutos
# Checa também pasta de spam!

# Volta o app
kubectl scale deployment sample-app --replicas=2
```

## 10. Debug

**Alertmanager não envia email:**

```bash
# Verifica config do Alertmanager
kubectl get secret alertmanager-config -n gmp-public -o yaml

# Checa logs
kubectl logs -n gmp-system -l app.kubernetes.io/name=alertmanager --tail=100

# Erros comuns:
# - "authentication failed" → App Password errada
# - "connection refused" → Firewall bloqueando porta 587
# - "invalid credentials" → Email errado ou App Password não habilitada
```

**Alertas não aparecem:**

```bash
# Verifica se rules foram criadas
kubectl get rules -A -o yaml

# Checa métricas manualmente (port-forward pro Prometheus)
kubectl port-forward -n gmp-system svc/frontend 9090:9090

# Acessa http://localhost:9090 e roda query: up{job="sample-app"}
```

**Email não chega:**

1. Checa pasta de spam
2. Confirma App Password do Gmail tá correta
3. Testa SMTP manualmente:
   ```bash
   kubectl run -it --rm debug --image=alpine --restart=Never -- sh
   apk add openssl
   openssl s_client -starttls smtp -connect smtp.gmail.com:587
   ```

**Alternativa: Usar outro provedor SMTP**

Gmail tem limite de 500 emails/dia. Alternativas:

- **SendGrid:** 100 emails/dia grátis
  ```yaml
  smtp_smarthost: 'smtp.sendgrid.net:587'
  smtp_auth_username: 'apikey'
  smtp_auth_password: 'SG.xxxxxx'
  ```

- **Mailgun:** 100 emails/dia grátis
  ```yaml
  smtp_smarthost: 'smtp.mailgun.org:587'
  smtp_auth_username: 'postmaster@seu.dominio'
  smtp_auth_password: 'sua-api-key'
  ```

- **AWS SES:** $0.10 por 1000 emails
  ```yaml
  smtp_smarthost: 'email-smtp.us-east-1.amazonaws.com:587'
  smtp_auth_username: 'AKIAXXXXX'
  smtp_auth_password: 'sua-smtp-password'
  ```

## Custos aproximados

- **GKE Autopilot:** ~$50-70/mês (2 pods pequenos)
- **GKE Standard:** ~$75/mês (2x e2-medium + control plane)
- **GMP (Prometheus):** FREE até 50GB métricas/mês
- **Gmail SMTP:** FREE até 500 emails/dia
- **Egress (email):** <$0.01/mês (tráfego mínimo)

**Total:** ~$50-75/mês (observabilidade é grátis!)

## Cleanup

```bash
# Deleta resources do cluster
kubectl delete -f k8s-manifests/

# Deleta cluster
gcloud container clusters delete $CLUSTER_NAME \
  --region=us-central1 \  # Autopilot
  --quiet

# OU (Standard)
gcloud container clusters delete $CLUSTER_NAME \
  --zone=us-central1-a \
  --quiet
```

## Diferenças do AWS

✅ **Vantagens GCP:**
- GMP totalmente gerenciado, zero config
- Alertmanager já integrado
- Observabilidade grátis (free tier permanente)
- Setup mais simples (sem SNS/Lambda)

❌ **Desvantagens:**
- Precisa configurar SMTP externo
- GKE um pouco mais caro que EKS

## Próximos passos

- Adiciona mais alertas (CPU, disk, etc)
- Configura rotas diferentes por severity
- Adiciona notificação via Slack/Discord
- Usa PagerDuty para on-call

Ver exemplos de alertas avançados em:
https://samber.github.io/awesome-prometheus-alerts/
