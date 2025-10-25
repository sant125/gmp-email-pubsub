# GCP Setup

## 1. Setup inicial

```bash
export PROJECT_ID="seu-project"
export ZONE="us-central1-a"

gcloud config set project $PROJECT_ID
gcloud config set compute/zone $ZONE

gcloud services enable container.googleapis.com \
  monitoring.googleapis.com \
  cloudpubsub.googleapis.com
```

## 2. Cria cluster

```bash
export CLUSTER_NAME="lab-cluster"

gcloud container clusters create $CLUSTER_NAME \
  --zone $ZONE \
  --num-nodes 2 \
  --machine-type e2-medium \
  --enable-managed-prometheus \
  --workload-pool=$PROJECT_ID.svc.id.goog

gcloud container clusters get-credentials $CLUSTER_NAME
kubectl get nodes
```

## 3. Configura notificações

```bash
# Email direto via Cloud Monitoring
gcloud alpha monitoring channels create \
  --display-name="Alertas Prometheus" \
  --type=email \
  --channel-labels=email_address=seu@email.com

# Lista pra pegar ID
gcloud alpha monitoring channels list

export CHANNEL_ID="projects/$PROJECT_ID/notificationChannels/XXXXXX"
```

## 4. Deploy app

```bash
kubectl apply -f k8s-manifests/sample-app.yaml
kubectl get pods
```

## 5. Configura coleta de métricas

```bash
kubectl apply -f k8s-manifests/pod-monitoring.yaml
kubectl get podmonitoring
```

## 6. Cria regras de alerta

```bash
kubectl apply -f k8s-manifests/alerting-rules.yaml
kubectl get rules -A
```

## 7. Configura alert policy

Vai em: https://console.cloud.google.com/monitoring/alerting

1. Create Policy
2. Add Condition → Prometheus Query Language
3. Query: `ALERTS{alertstate="firing"}`
4. Duration: 60s
5. Add Notification Channel → seleciona teu email
6. Salva

## 8. Testa

```bash
# Dispara alerta
kubectl scale deployment sample-app --replicas=0

# Aguarda 2 min, checa email

# Volta
kubectl scale deployment sample-app --replicas=2
```

## Diferenças do AWS

- GMP é automático, não precisa instalar agent
- PodMonitoring é mais simples
- Observabilidade grátis no free tier
- Alert policies pelo Cloud Console

Ver troubleshooting no README principal.
