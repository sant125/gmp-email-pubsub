# AWS Setup

## 1. Cria cluster

```bash
export CLUSTER_NAME="lab-cluster"
export AWS_REGION="us-east-1"

eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --nodes 2 \
  --node-type t3.medium \
  --with-oidc \
  --managed

kubectl get nodes
```

## 2. Cria workspace AMP

```bash
aws amp create-workspace \
  --alias $CLUSTER_NAME-prometheus \
  --region $AWS_REGION

export WORKSPACE_ID=$(aws amp list-workspaces \
  --alias $CLUSTER_NAME-prometheus \
  --query 'workspaces[0].workspaceId' \
  --output text)
```

## 3. Setup SNS

```bash
aws sns create-topic --name prometheus-alerts --region $AWS_REGION

export TOPIC_ARN=$(aws sns list-topics \
  --query "Topics[?contains(TopicArn, 'prometheus-alerts')].TopicArn" \
  --output text)

aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol email \
  --notification-endpoint seu@email.com
  
# Confirma o email!
```

## 4. IAM Setup

```bash
# Policy SNS
cat > amp-sns-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["sns:Publish"],
    "Resource": "$TOPIC_ARN"
  }]
}
EOF

aws iam create-policy \
  --policy-name AMPAlertManagerSNSPolicy \
  --policy-document file://amp-sns-policy.json

# Anexa na role do cluster
export CLUSTER_ROLE=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query "cluster.roleArn" \
  --output text | cut -d'/' -f2)

aws iam attach-role-policy \
  --role-name $CLUSTER_ROLE \
  --policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/AMPAlertManagerSNSPolicy

# IRSA pro Prometheus
kubectl create namespace prometheus

eksctl create iamserviceaccount \
  --name amp-iamproxy-ingest \
  --namespace prometheus \
  --cluster $CLUSTER_NAME \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonPrometheusRemoteWriteAccess \
  --approve
```

## 5. Instala Prometheus

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Edita prometheus-values.yaml com teu WORKSPACE_ID
helm install prometheus-for-amp prometheus-community/prometheus \
  --namespace prometheus \
  -f prometheus-config/prometheus-values.yaml

kubectl get pods -n prometheus
```

## 6. Deploy app

```bash
kubectl apply -f k8s-manifests/sample-app.yaml
kubectl get pods
```

## 7. Configura alertas

```bash
# Upload rules
cat prometheus-config/alerting-rules.yaml | base64 > /tmp/rules.txt
aws amp create-rule-groups-namespace \
  --workspace-id $WORKSPACE_ID \
  --name lab-alerts \
  --data fileb:///tmp/rules.txt

# Upload alertmanager (edita o TOPIC_ARN antes)
cat prometheus-config/alertmanager-config.yaml | base64 > /tmp/am.txt
aws amp create-alert-manager-definition \
  --workspace-id $WORKSPACE_ID \
  --data fileb:///tmp/am.txt
```

## 8. Testa

```bash
# Dispara alerta
kubectl scale deployment sample-app --replicas=0

# Aguarda 2 min, checa email

# Volta
kubectl scale deployment sample-app --replicas=2
```

Ver troubleshooting no README principal.
